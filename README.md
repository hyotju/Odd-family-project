# 결과물 
  https://www.instagram.com/p/DYy5uublPd-/?igsh=MTNndWc4cTZxNDNhaA==
# notion 페이지
  https://app.notion.com/p/Odd-Family-Project-320d018677e3807c84e9d7046629f2fa
  
  마음 부적
  - https://app.notion.com/p/225dabec87de82ea92a481c75a37758d
  - https://www.figma.com/design/csOEbKyCb7pByqJEn1FiMm/%EB%A7%88%EC%9D%8C%EB%B6%80%EC%A0%81?node-id=104-2&p=f&m=dev
  

# 4-chatbot

HateSlop **4기** 세션용 워크스페이스입니다. Flask + RAG 캐릭터 챗봇 실습을 위한 **기본 템플릿**과, 이전 기수 **참고 예시**를 한 저장소에서 다룹니다.

## 폴더 구성

| 경로 | 역할 |
|------|------|
| [**4-base/**](4-base/) | 4기 엔지니어×프로듀서 합동 캐릭터 챗봇 **과제 베이스**. 여기서 개발·실행합니다. |
| [**example/**](example/) | 2기·3기 예시 저장소 링크 모음. 로컬에 복사해 둔 샘플 프로젝트가 있을 수 있습니다. |

## 빠른 시작 (4-base)

실행·Docker·가상환경 절차는 **`4-base/README.md`**에 정리되어 있습니다.

```bash
cd 4-base
cp .env.example .env   # OPENAI_API_KEY 입력
```

- **Docker**: `4-base`에서 `docker compose up --build` → 브라우저 [http://localhost:5001](http://localhost:5001)
- **로컬 가상환경**: 같은 문서의 「가상환경으로 로컬 실행」 참고 → 기본 [http://127.0.0.1:5000](http://127.0.0.1:5000)

## example 참고

- [example/README.md](example/README.md) — 2기 공식 예시·3기 조별 원본 GitHub
- [example/실행방법.md](example/실행방법.md) — `example/` 안 각 프로젝트 실행 방법

## 문서 더 보기

- [4-base/README.md](4-base/README.md) — 목표, 아키텍처, 배포 가이드 링크
- [4-base/ARCHITECTURE.md](4-base/ARCHITECTURE.md), [4-base/DOCKER-GUIDE.md](4-base/DOCKER-GUIDE.md) 등

---

*이 루트(`4-chatbot`)는 멀티 프로젝트 컨테이너가 아닙니다. 앱은 항상 **`4-base`**(또는 `example/`의 개별 폴더) 단위로 실행합니다.*
