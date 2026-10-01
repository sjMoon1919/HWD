# 해운대 데스크

해운대구 1인 의정자료 작업 공간. Next.js 16 + TypeScript + Supabase, GitHub/Vercel 배포용 소스입니다. 파란 카드와 민트·핑크 폴더, 탐색기 창, 작업 표시줄을 사용합니다.

## 바로 실행

Node.js 22 이상을 설치한 PC에서 이 폴더를 터미널로 엽니다.

```powershell
npm install
npm run dev
```

http://127.0.0.1:3000 을 엽니다. 키 없이 실행하면 **예시 모드**입니다. 예시 수치는 실제 해운대구 예산이 아닙니다. 예시 모드에서 가져온 문서는 메모리에만 있으며 새로고침 시 사라집니다. 설정 연결 이후 새 문서를 업로드해야 영구 저장됩니다. 개인 메모는 이 브라우저 localStorage에 저장합니다.

## Supabase 연결 (1인 전용)

1. Supabase 프로젝트를 만듭니다. Auth → Users → Add user에서 본인 이메일 계정을 직접 만듭니다. Auth 설정의 새 사용자 가입을 비활성화합니다.
2. 생성된 사용자 UUID를 복사합니다. `supabase/schema.sql`의 `00000000-0000-0000-0000-000000000000`을 그 UUID로 교체합니다. SQL Editor에서 `schema.sql`, 이어서 `reviews.sql`을 한 번씩 실행합니다. 빈 프로젝트에 적용하는 초기 스키마이므로 재실행하면 기존 테이블 오류가 납니다.
3. `.env.example`을 `.env.local`로 복사합니다. 프로젝트 URL과 anon 공개 키를 입력합니다. **service_role 키는 필요 없으며 브라우저에 넣지 마세요.**
4. 앱을 다시 시작하고 설정에서 로그인합니다. 공개 회원가입 화면은 없으며, RLS와 Storage 정책은 등록한 소유자 UUID에게만 허용합니다.
5. PDF/XLSX를 가져옵니다. 원문 페이지와 후보를 대조해 사업명·원 단위 금액·계층을 수정하고 확정합니다. 한 문서에서 같은 집계 대상의 상위 총액 행과 하위 사업 행을 동시에 확정하면 중복 합산되므로 분석 대상 사업 행만 확정하세요.

DB/Storage 연동은 계정 연결 전이라 이 전달본에서 실제 클라우드 검증을 수행하지 않았습니다. 배포 전에 본인 계정의 저장/재로그인/원본 열기와 별도 계정의 접근 거부를 확인하세요.

## 문서와 분석 흐름

`원본 직접 업로드 → SHA-256 중복 확인 → PDF 텍스트 추출 / 스캔 페이지만 한국어·영어 OCR → 페이지·시트 텍스트와 사업 후보 저장 → 원문 대조·확정 → 코드 집계·사업 계보 후보 → 필요한 구조화 데이터만 AI에 전송 → 근거와 데이터 스냅샷 보존`

- PDF.js가 PDF를 페이지별로 읽습니다. 텍스트가 거의 없는 페이지는 Tesseract 한국어/영어 OCR을 실행합니다. 최초 OCR 사용 시 CDN에서 엔진·언어 자료를 다운로드합니다. 문서 이미지 자체는 OCR 서버로 전송하지 않으며 브라우저에서 처리합니다. 암호화 PDF와 손상 파일은 지원하지 않습니다.
- ExcelJS가 `.xlsx` 시트별 텍스트를 읽습니다. `.xls`는 Excel에서 `.xlsx`로 변환하세요. Excel 근거의 페이지 번호는 시트 순번이며, 인쇄 페이지 번호가 아닙니다.
- OCR/표 추출은 후보 추출입니다. 모든 문서 레이아웃, 병합 셀, 다중 금액 열, 계층 자동 인식을 보장하지 않습니다. 첫 숫자 열을 후보로 잡으므로 반드시 열·단위를 대조합니다. 천원은 원으로 변환해 입력합니다. 자동 계층 값은 비워 두고 사용자가 원문 기준으로 입력합니다. 원본이 보존되어 수동 사업 추가도 가능합니다.
- 본예산·추경·결산·사업명세서·기금·검토보고서를 분류하고 연도를 부여합니다. 3년 합계는 확정된 사업만 대상으로 계산합니다. 자료 없는 연도는 0원과 0개 사업으로 표시되므로 실제 무예산으로 해석하지 마세요. 전년도 값이 0이면 증감률은 계산하지 않습니다.
- 분석은 문서 종류를 하나 선택합니다. 같은 해·같은 종류의 문서가 여러 개면 모두 합산됩니다. 분석할 문서 선택에서 비교에 넣을 문서를 체크하세요. 현재 버전은 회계/추경 차수/중복 총액 제거를 자동 판정하지 않습니다.
- 사업 계보는 t-2→t-1 및 t-1→t 연도의 후보를 제시합니다. 사업명 60%, 금액 20%, 동일 계층 20%로 점수를 만들며, 45% 이상 최대 3개 후보를 표시합니다. 분할·통합·동일 사업 여부의 최종 확정은 사용자의 검토 영역입니다. 계보는 자동 추론이며 공식 식별자가 아닙니다.
- 문서는 `documents`, 페이지/시트 텍스트는 `document_pages`, 확정 사업은 `budget_rows`, AI/계산 검토 결과와 당시 데이터·근거는 `reviews`에 저장합니다. 페이지 근거는 외래키로 제한합니다. `save_document` RPC는 문서와 구조화 데이터를 트랜잭션으로 저장합니다. 원본은 비공개 Storage에 보관하고 10분 유효한 서명 URL로 열 수 있습니다.
- 반복 분석에서는 PDF/OCR을 호출하지 않습니다. Supabase에서 불러온 구조화 행을 코드로 합산·필터링합니다. AI API에는 선택한 확정 행과 코드 계산, 문서 ID·페이지 근거만 전송합니다. 원문 PDF나 페이지 전체 텍스트는 보내지 않습니다. 문서는 데이터로 취급하도록 프롬프트를 구성합니다.
- AI 검토는 초안입니다. 제공된 근거 번호를 원문으로 열어 검증하고 사용합니다. 계정 연결 시 `reviews`에 숫자·출처 스냅샷을 보관하고, 화면에서는 근거 목록을 포함한 TXT 다운로드를 제공합니다. 과거 검토 목록 UI는 아직 별도로 구현하지 않았으며 Supabase의 `reviews`에서 확인할 수 있습니다.
- 행정사무감사·5분 자유발언·조례안 폴더는 향후 확장용 개인 메모 공간입니다. 전용 문서 생성 기능은 포함되지 않았습니다.

## AI 연결 (선택)

`.env.local` 또는 Vercel 환경 변수에 `OPENAI_API_KEY`를 넣습니다. `OPENAI_MODEL`로 계정에서 사용 가능한 모델을 선택할 수 있습니다. 키가 없거나 응답 실패 시 코드 계산 기반 초안을 표시합니다. `/api/review`는 Supabase 사용자 인증 및 RLS를 사용해 서버에서 자료를 조회합니다. 임의 클라이언트 금액을 그대로 믿지 않습니다. 300개 이하의 확정 사업 범위만 분석하며 그보다 많으면 범위를 좁히라는 오류를 반환합니다. AI 응답은 45초 타임아웃을 둡니다. AI 키는 GitHub에 올리지 않습니다.

## GitHub → Vercel

1. GitHub에 빈 저장소를 만듭니다. 이 폴더의 소스를 올립니다. `node_modules`, `.next`, `.env.local`, `work`는 제외합니다. 폴더를 그대로 저장소 루트로 사용하세요.
2. 터미널을 이 폴더에서 열어 아래 명령을 실행합니다. `YOUR_REPOSITORY_URL`은 본인의 실제 URL로 바꾸세요.

```powershell
git init
git add .
git commit -m "Build Haeundae personal desk"
git branch -M main
git remote add origin YOUR_REPOSITORY_URL
git push -u origin main
```

3. Vercel → Add New Project → 해당 GitHub 저장소를 연결합니다. Framework는 Next.js, `vercel.json`에서 pnpm 11.19.0 설치·빌드 명령을 지정했습니다. 이 전달본에 `pnpm-lock.yaml`이 있어 pnpm 설치를 재현할 수 있습니다. `pnpm-workspace.yaml`은 esbuild/tesseract 설치 스크립트를 허용합니다.
4. Vercel 환경 변수에 `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`와 선택 AI 키를 입력합니다. 서버 함수에 AI를 사용할 때 플랫폼의 실행 시간 제한을 확인하세요.
5. Deploy 후 배포 URL에서 로그인·업로드·OCR·확정·원본 열기·재로그인 후 자료 보존을 확인합니다. 이후 GitHub main 브랜치 변경은 Vercel 배포로 연결됩니다.

이 전달본은 실제 GitHub 저장소 생성/푸시 및 Vercel 배포를 하지 않았습니다. 저장소 URL과 계정 연결이 있어야 수행할 수 있습니다.

## Windows 바탕화면에서 실행

**추천:** HTTPS 배포 URL을 Edge에서 엽니다. 메뉴 → 앱 → 이 사이트를 앱으로 설치 → 바탕화면 바로가기 생성을 선택합니다. Chrome에서도 주소창 설치 아이콘 또는 앱 설치 메뉴를 사용합니다. 설치 후 독립 창으로 열립니다. PWA이며 Windows `.exe` 패키지가 아닙니다. 자료 조회와 DB 저장은 온라인 연결이 필요합니다. 서비스 워커는 개인 문서나 인증 응답을 캐시하지 않습니다.

일반 웹 바로가기는 아래 스크립트로 만들 수 있습니다. 주소를 실제 Vercel 주소로 변경하세요.

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\create-desktop-shortcut.ps1 -Url "https://YOUR-PROJECT.vercel.app"
```

## 검증

```powershell
npm run typecheck
npm test
npm run build
```

전달 시 TypeScript/배포 빌드 및 계산·금액검증·출처 보존 테스트 5개 통과. 브라우저에서 시험 XLSX·텍스트 PDF·한국어 스캔 PDF 업로드와 OCR, 사업 확정, 검토 근거 페이지 이동을 확인했습니다. 클라우드 계정과 실제 해운대구 예산 문서에 대한 검증은 연결 후 수행해야 합니다. 큰 스캔 PDF는 브라우저 메모리와 처리 시간이 필요하므로 업로드 중 탭을 닫지 마세요.

공식 참고: [Next.js 설치](https://nextjs.org/docs/app/getting-started/installation), [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security), [Storage 접근 제어](https://supabase.com/docs/guides/storage/security/access-control), [Vercel 함수 제한](https://vercel.com/docs/functions/limitations).
