Tikke 배포 또는 릴리즈 전 상태를 점검한다.

절차:
1. 릴리즈 대상이 데스크탑, 웹, Worker 중 무엇인지 확인한다.
2. `git status`로 변경 상태를 확인한다.
3. 대상별 검증 명령어를 실행하거나 계획한다.
   - 데스크탑: `pnpm build:desktop`, 필요 시 `pnpm dist:win`
   - 웹: `pnpm --filter @tikke/web build`
   - Worker: `pnpm --filter @tikke/worker typecheck`
   - 전체: `pnpm typecheck`, 필요 시 `pnpm build`
4. 릴리즈 위험 요소와 남은 작업을 정리한다.

주의:
- 사용자가 명시적으로 요청하지 않는 한 `git push`, `gh release create`, `wrangler deploy`는 실행하지 않는다.
- secret 값을 출력하지 않는다.

요청:
$ARGUMENTS
