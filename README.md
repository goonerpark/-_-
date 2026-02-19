
# drawing-navigator

## 실행 방법
cd drawing-navigator
npm install
npm run dev

## 기술 스택
- Vue 3
- TypeScript
- Vite
- HTML/CSS (Vanilla)

## 구현 기능
- metadata.json 기반 도면 데이터 로딩
- 도면 데이터 트리 구조 렌더링 (프로젝트 → 도면 → 공종 → 리전 → 리비전)
- 트리에서 선택한 항목을 우측 뷰어에 표시
- 리비전 상세 정보 표시 (버전/날짜/설명/변경사항)
- 공종 간 간섭 확인용 오버레이 모드
  - 같은 drawingId 내 후보 선택
  - 투명도 조절
  - 미세 조정(dx/dy/scale/rotation)
  
## 미완성 기능
- 몇몇 도면에서 오버레이기능을 사용할 때, 도면이 겹쳐지지 않는 오류 발생
- 검색/필터 기능
- 오버레이 자동 정렬 정확도 개선 필요
