# Retrieval API

## 목적

사용자 질문에 대한 답변과 근거 슬라이드 반환

## 엔드포인트

| Method | Path | 목적 | 요청 | 응답 | 인증 |
|---|---|---|---|---|---|
| `POST` | `/workspaces/{workspaceId}/queries` | 질문 실행 | question, document filters, topK | answer, evidences | 조회권한 |
| `GET` | `/queries/{queryLogId}` | 질의 이력 조회 | 없음 | 질문/답변/근거 | 조회권한 |
| `GET` | `/documents/{documentId}/slides/{slideNo}` | 슬라이드 근거 상세 조회 | 없음 | summary, chunks, thumbnail | 조회권한 |
| `GET` | `/documents/{documentId}/slides/{slideNo}/workflow` | 워크플로우 추출 결과 조회 | 없음 | nodes, edges, summary | 조회권한 |
## 질의 요청 예시

```json
{
  "question": "2024년 성장률이 가장 높은 사업 부문은 무엇인가?",
  "documentIds": [101, 102],
  "topK": 8,
  "includeOcr": true,
  "queryTypeHint": "AUTO",
  "allowNoAnswer": true
}
```

## 질의 응답 예시

```json
{
  "data": {
    "answer": "2024년 성장률이 가장 높은 사업 부문은 플랫폼 사업으로 보인다.",
    "evidences": [
      {
        "documentId": 101,
        "documentName": "annual-report.pptx",
        "slideNo": 12,
        "chunkType": "SUMMARY",
        "snippet": "플랫폼 사업의 성장률이 35%로 가장 높음",
        "score": 0.93
      }
    ]
  }
}
```

## 응답 규칙

- `evidences`는 slide 기준으로 중복 축약 가능
- 답변 생성에 사용된 상위 슬라이드는 최대 5개 권장
- score는 내부값이지만 디버깅과 운영을 위해 노출 가능
- workflow 질의로 판정되면 `WORKFLOW` chunk를 우선 재랭킹한다
- 근거 부족 시 `allowNoAnswer=true`이면 추정 답변보다 no-answer를 우선한다
