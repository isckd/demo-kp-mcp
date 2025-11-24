# demo-kp-mcp
demo-kp-mcp

```python
import json
import re
from html import unescape
from typing import Any, Dict, Iterable, List, Optional

import truststore
import requests


BASE_PAGE_URL = "https://www.konaplate.com/developers/api-documentation"
DATA_URL_TEMPLATE = (
    "https://www.konaplate.com/_next/data/{build_id}/ko/developers/api-documentation.json"
)
NEXT_DATA_SCRIPT_REGEX = re.compile(
    r'<script id="__NEXT_DATA__" type="application/json">(.+?)</script>', re.S
)
HTML_TAG_REGEX = re.compile(r"<[^>]+>")


def fetch_build_id(session: requests.Session) -> str:
    truststore.inject_into_ssl()
    response = session.get(BASE_PAGE_URL, timeout=15)
    response.raise_for_status()

    match = NEXT_DATA_SCRIPT_REGEX.search(response.text)
    if not match:
        raise RuntimeError("__NEXT_DATA__ 스크립트를 찾을 수 없습니다.")

    payload = json.loads(match.group(1))
    build_id = payload.get("buildId")
    if not build_id:
        raise RuntimeError("buildId 정보를 찾을 수 없습니다.")

    print(f"Fetched buildId: {build_id}")
    return build_id


def fetch_api_documentation(session: requests.Session, build_id: str) -> Dict[str, Any]:
    data_url = DATA_URL_TEMPLATE.format(build_id=build_id)
    response = session.get(data_url, timeout=15)
    response.raise_for_status()
    return response.json()


def clean_html(value: Optional[str]) -> str:
    if not value:
        return ""
    text = HTML_TAG_REGEX.sub(" ", value)
    text = unescape(text)
    return re.sub(r"\s+", " ", text).strip()


def pick_localized_text(descriptions: Optional[Iterable[Dict[str, Any]]]) -> str:
    if not descriptions:
        return ""
    preferred_langs = ("KO", "ko", "KR", "kr", "EN", "en", None)
    for lang in preferred_langs:
        for entry in descriptions:
            if entry.get("lang") == lang:
                return clean_html(entry.get("content"))
    first = next(iter(descriptions), None)
    return clean_html(first.get("content")) if first else ""


def summarize_length(length_spec: Optional[Dict[str, Any]]) -> Optional[str]:
    if not length_spec:
        return None
    spec_type = length_spec.get("type")
    if spec_type == "fixed":
        return str(length_spec.get("value"))
    if spec_type == "range":
        return f"{length_spec.get('min')}-{length_spec.get('max')}"
    return None


def format_params(params: Optional[List[Dict[str, Any]]]) -> List[Dict[str, Any]]:
    formatted: List[Dict[str, Any]] = []
    for param in params or []:
        entry = {
            "field": param.get("field"),
            "type": param.get("type"),
            "moc": param.get("moc"),
            "length": summarize_length(param.get("length")),
            "description": pick_localized_text(param.get("description")),
        }
        children = format_params(param.get("params"))
        if children:
            entry["children"] = children
        entry = {k: v for k, v in entry.items() if v}
        if entry:
            formatted.append(entry)
    return formatted


def extract_group_path(page_props: Dict[str, Any]) -> List[str]:
    pre_selected = page_props.get("preSelectedApi", {})
    path_nodes = pre_selected.get("path", [])
    titles = [node.get("title") for node in path_nodes if node.get("title")]
    node_title = pre_selected.get("node", {}).get("title")
    if node_title:
        titles.append(node_title)
    return titles


def extract_response_fields(response_codes: Optional[List[Dict[str, Any]]]) -> List[Dict[str, Any]]:
    for code in response_codes or []:
        params = code.get("resParam")
        if params:
            return format_params(params)
    return []


def extract_response_codes(response_codes: Optional[List[Dict[str, Any]]]) -> List[Dict[str, Any]]:
    summarized = []
    for code in response_codes or []:
        reason = code.get("reasonCodeJs", {})
        description = pick_localized_text(reason.get("description"))
        summarized.append(
            {
                "httpCode": code.get("httpCode"),
                "code": reason.get("reason"),
                "message": reason.get("message"),
                "description": description,
            }
        )
    return summarized


def summarize_api_spec(api_doc: Dict[str, Any]) -> Dict[str, Any]:
    page_props = api_doc.get("pageProps", {})
    api_info = page_props.get("apiInfoResponse")
    if not api_info:
        raise RuntimeError("apiInfoResponse 정보를 찾을 수 없습니다.")

    summary: Dict[str, Any] = {
        "categoryPath": " > ".join(extract_group_path(page_props)) or None,
    }

    api_details: Dict[str, Any] = {
        "title": api_info.get("title"),
        "description": api_info.get("description"),
        "method": api_info.get("method"),
        "url": api_info.get("apiUrl"),
        "protocol": api_info.get("protocol"),
        "version": api_info.get("apiVersion"),
        "status": api_info.get("infoStatus"),
        "encryption": {
            "request": api_info.get("reqEncrypt") == "Y",
            "response": api_info.get("resEncrypt") == "Y",
        },
    }

    headers = format_params(api_info.get("reqHeaderJson"))
    if headers:
        api_details["requestHeaders"] = headers

    request_params = format_params(api_info.get("reqParamJson"))
    if request_params:
        api_details["requestParams"] = request_params

    response_fields = extract_response_fields(api_info.get("responseCode"))
    if response_fields:
        api_details["responseFields"] = response_fields

    response_codes = extract_response_codes(api_info.get("responseCode"))
    if response_codes:
        api_details["responseCodes"] = response_codes

    summary["api"] = {k: v for k, v in api_details.items() if v is not None}
    return summary


def main() -> None:
    session = requests.Session()
    session.headers.update({"User-Agent": "Mozilla/5.0 (Cascade Agent)"})

    build_id = fetch_build_id(session)
    api_doc = fetch_api_documentation(session, build_id)
    summary = summarize_api_spec(api_doc)
    print(json.dumps(summary, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```
