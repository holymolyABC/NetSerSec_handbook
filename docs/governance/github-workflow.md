# GitHub 저장소 등록과 문서 운영

## 현재 저장소

- 주소: [holymolyABC/NetSerSec_handbook](https://github.com/holymolyABC/NetSerSec_handbook)
- 이름: `NetSerSec_handbook`
- 공개 범위: **Public** — 2026-10-15 업로드 전 확인
- 기본 브랜치: `main`
- 문서 첫 화면: 루트 `README.md`

사용자가 지정한 공개 저장소에 맞춰 고객 도메인·IP를 예제 값으로 치환했습니다. 원본 설정·개인정보·인증 키는 포함하지 않습니다. 실제 운영 정보가 필요한 문서는 별도 접근 통제된 저장소에서 관리합니다. 공개 여부는 향후 변경될 수 있습니다.

일반 Markdown으로 되어 있어 GitHub에서 바로 읽을 수 있습니다. 별도 빌드나 GitHub Pages 배포 없이 분류별 README의 상대 링크로 탐색합니다.

## 웹에서 직접 업로드하는 경우

1. 패키지 ZIP을 압축 해제합니다.
2. `infra-ops-handbook` 폴더의 **내용물**을 저장소 최상위로 업로드합니다. 폴더 자체가 한 단계 중첩되지 않도록 합니다.
3. 초기 README가 있으면 본 핸드북의 README 내용으로 교체합니다.
4. ZIP 파일만 업로드하면 내부 문서가 GitHub 목차로 펼쳐지지 않습니다. 압축 해제한 문서를 올립니다.
5. 커밋 후 README의 상대 링크와 examples 파일이 열리는지 확인합니다.

## 이미 Git을 사용하는 PC에서 올리는 경우

빈 저장소와 초기 README가 있는 저장소의 이력이 다르므로 무조건 force push하지 않습니다. 일반적인 방법은 새 저장소를 clone하고 **그 안으로** 압축 해제한 파일을 복사하는 것입니다.

```bash
# Git과 인증이 준비된 PC에서만 실행
git clone https://github.com/holymolyABC/NetSerSec_handbook.git
cd NetSerSec_handbook
# 이 디렉터리에 압축 해제한 문서 내용물을 복사한 후
git status --short
git diff
git add README.md CONTRIBUTING.md CHANGELOG.md REFERENCES.md .gitignore docs templates examples
git diff --cached --stat
git diff --cached
git commit -m "docs: initialize infrastructure operations handbook"
git push origin main
```

공유 전 고객 도메인·IP·개인정보·키·토큰이 포함되지 않았는지 확인합니다. 인증 토큰을 원격 URL에 넣거나 채팅으로 전달하지 않습니다.

## 이후 확장

새 작업은 기존 매뉴얼 검색 → 변경 기록 작성 → 검증 증거 기록 → 재사용 절차로 추출 → 분류 README 링크 추가 → commit 순서로 관리합니다. 협업하면 작업별 브랜치와 PR을 사용합니다.

| 문서 수정 | 커밋 예시 |
|---|---|
| DNS 레코드 등록 절차 수정 | `docs(dns): clarify zone reload verification` |
| 장비별 스위치 점검 추가 | `docs(network): add model-specific read-only checks` |
| 방화벽 변경 사례 추가 | `docs(case): record firewall policy validation` |

최초 문서 제출만으로 명령이 모든 버전에서 검증된 것은 아닙니다. 실제 검증일과 대상 버전을 계속 갱신합니다.
