# Streamlit 통합 검증 공유

기준일: 2026-08-31  
통합 브랜치: `feat/chain`

## 통합한 범위

- 기존 `main` 기준 카탈로그·RAG·문서 파이프라인·Streamlit 작업을 유지했다.
- 2026-08-31 `origin/fix/dev` 최신 6개 커밋(`c06aabb`∼`7ac76ea`)을 fast-forward로 반영했다.
- 메인에 아직 없던 문서 정제·청킹 품질 게이트(`4ff58c0`, 원격 PR 병합 `4708e91`)와 RAG 계약·adapter 변경을 추가 통합했다.
- Streamlit mock 체인을 실제 `RagQaService`와 `RecommendationRagService`로 교체했다.
- 문서 품질 계약의 `quality_status`·`embedding_checksum`을 manifest→RAG→공통 검색 응답에 끝까지 전파했고, BM25도 검수 승인 청킹만 공식 근거로 사용하도록 맞춰서 병합 후 계약 불일치를 해소했다.
- 제품 추천 폼은 `RecommendationFormInput`으로 검증하고, 위젯에서 선택한 명시적 조건이 sLLM 결과보다 우선되도록 `answer_form()` 진입점을 추가했다.
- QA·추천 결과는 공통 `ChatResponse 1.1.0`의 답변, 조건, 제품 카드, 인용 카드, 보류 상태를 그대로 표시한다.
- 실행 파일이 누락된 경우 mock으로 숨기지 않고 준비 필요 상태를 화면에 보여 준다.

## 현재 연결 흐름

```text
Streamlit 입력
  ├─ QA → RagQaService → HybridRetriever → AnswerGenerator → 인용 검증
  └─ 제품 추천 → RecommendationFormInput → sLLM 조건 추출
                → catalog 필터 → 후보 문서 제한 Hybrid RAG
                → AnswerGenerator → ChatResponse
```

Streamlit은 `streamlit_app/runtime.py`에서 CLI와 동일한 `.env`를 읽어 검색기와 답변 생성기를 조립한다. 제품·URL·이미지·출처 카드는 LLM 자유 출력이 아니라 catalog와 manifest metadata에서 구성된다.

## 검증 결과

| 항목 | 결과 |
|---|---|
| 전체 자동 테스트 | `158 passed in 0.62s` |
| Streamlit 문법·import | `py_compile` 통과 |
| Streamlit AppTest | 예외 0건, 런타임 준비 안내 표시 확인 |
| 실제 서버 기동 | `streamlit run streamlit_app/app.py --server.headless true --server.port 8502` 성공 |
| health endpoint | `GET /_stcore/health` → `ok` |
| 첫 화면 HTTP | `/` 응답 정상 |
| Streamlit 폼 계약 테스트 | 명시 위젯값 우선·`ChatResponse` 반영 통과 |

## 현재 남은 실행 자료

코드 연결과 UI 기동은 통과했지만, 현재 작업 폴더에 아래 산출물이 없어 실제 질문→최종 답변 E2E는 아직 실행할 수 없다.

1. 프로젝트 루트의 `.env` (`.env.example` 기준)
2. 검수된 `document_pipeline/data/manifest_v3.json`
3. 같은 manifest로 생성한 `data/indexed/chroma_official_v3/` 전체
4. 제품 추천용 LoRA adapter 로컬 경로 또는 비공개 Hub ID·접근 권한
5. RunPod 실제 생성을 사용할 경우 Qwen 모델과 CUDA 환경

정식 manifest는 로컬 smoke용 임시 문서로 대체하지 않는다. 담당자별 전달 기준은 [`docs/data-contracts/team-handoff.md`](../data-contracts/team-handoff.md)를 따른다.

## 산출물 수령 후 E2E 실행 순서

```bash
cp .env.example .env
# .env의 LORA_ADAPTER_PATH 등을 실제 실행 환경에 맞게 수정
python -m src.services.rag_qa_cli --action index
python -m src.services.rag_qa_cli --query "SSH를 활성화하려면?" --trace
python -m src.services.recommendation_rag_cli --query "모니터 없이 홈 서버로 쓸 보드를 추천해줘" --trace
streamlit run streamlit_app/app.py
```

실행 후 제품 추천에서 조건 JSON·제품 카드·인용 카드를, QA에서 답변의 `[C1]` 인용과 출처 카드 일치를 확인한다. 근거 부족·범위 외·안전 차단 질문은 제품 카드나 출처를 임의 표시하지 않아야 한다.
