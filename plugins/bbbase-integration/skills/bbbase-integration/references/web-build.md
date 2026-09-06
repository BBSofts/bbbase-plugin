# 웹(브라우저) 빌드 연동 — CORS origin + 게스트 로그인

게임이 **브라우저에서 도는 빌드**면 이 문서를 먼저 읽는다: 순수 JS/TS 웹게임, Godot Web/WASM,
Unity WebGL, 앱인토스 미니앱 등. 네이티브 빌드(Unity/Godot 데스크톱·모바일)는 브라우저가
아니므로 이 문서와 무관하다 — 그쪽은 `references/game-auth.md` 만 보면 된다.

인증 헤더·envelope·compareMode 규약은 다른 플랫폼과 100% 동일하다. **웹에만 추가로 필요한
것은 origin(CORS) 등록 하나**다.

## 1. 🚨 origin 등록이 선행 필수 — 코드보다 먼저

등록 전에는 **로그인 요청 자체가 브라우저에서 차단**된다. 서버까지 가더라도 응답을 읽을 수
없으므로 어떤 코드로도 우회할 수 없다.

```bash
bbbase origin:add {PROJECT_ID} --origin https://my-game.example.com
bbbase origin:list {PROJECT_ID}
bbbase origin:remove {PROJECT_ID} --origin https://old.example.com
```

대시보드에서는 **프로젝트 → 소셜 로그인 설정 → 허용 Origin (CORS)** 카드에서 같은 일을 한다.
서버 재배포 없이 즉시 반영된다(운영자 JWT 필요).

- **스킴+호스트+포트까지 정확히** 일치해야 한다: `https://a.com` ≠ `http://a.com` ≠
  `https://a.com:8443`. 경로(`/game`)는 origin 이 아니므로 넣지 않는다.
- **로컬 개발 주소도 따로 등록**한다(예 `http://localhost:5173`). 배포 origin 만 등록하면
  개발 내내 막힌다.
- 운영자 권한이 없으면 값을 추측하지 말고 **개발자에게 등록을 요청**한다.

## 2. CORS 에러 진단 — 코드를 고치지 말 것

`blocked by CORS policy` / `No 'Access-Control-Allow-Origin' header` 는 **BBBase 에러코드가
아니다.** 응답 자체가 도달하지 않아 `error.code` 가 존재하지 않는다. 따라서:

- ❌ `fetch` 옵션을 바꾸거나 헤더를 빼서 해결하려 들지 말 것
- ❌ `mode: 'no-cors'` 금지 — 요청은 나가지만 응답을 읽을 수 없어 로그인이 영원히 실패한다
- ❌ 프록시를 새로 세우는 것도 첫 수단이 아니다
- ✅ **§1 의 origin 등록 여부부터 확인**한다. 현재 페이지 origin 은 브라우저 콘솔에서
  `location.origin` 으로 확인해 그 값을 그대로 등록한다

401/403 은 반대로 정상적인 BBBase 응답이므로 `references/game-auth.md` 의 인증 규약 문제다.

## 3. 게스트 로그인 — `deviceId` 에 무엇을 넣나

요청 본문은 **`deviceId`(문자열, 필수, 최대 256자) 하나**다. 서버는 이 값을 게스트 신원의
식별키로 삼아 find-or-create 한다 — **같은 값 = 같은 `userId` = 같은 세이브**. 즉 `deviceId`
는 "잃으면 계정을 잃는" 값이다.

```js
const res = await fetch(`${BASE_URL}/projects/${PROJECT_ID}/auth/guest`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-API-Key': API_KEY },
  body: JSON.stringify({ deviceId }),
});
const { success, data } = await res.json();  // data = { accessToken, refreshToken, userId }
```

무엇을 넣을지는 플랫폼이 정한다 — **더 안정적인 값이 있으면 그걸 쓴다**:

| 환경 | 권장 `deviceId` |
|---|---|
| 앱인토스 미니앱 | `User.getAnonymousKey()` 의 **앱 스코프 익명키(hash)** — 로그인 프롬프트 없이 안정적 |
| 일반 웹 | `localStorage` 에 보관한 `crypto.randomUUID()` (없으면 생성해 저장) |
| 네이티브 | 기기 ID / SDK 가 관리하는 값 |

> ⚠️ **웹의 `localStorage` 는 잘 날아간다**(캐시 삭제·시크릿 모드·다른 브라우저/기기). 게스트
> 계정도 그만큼 쉽게 유실되니, 진행도가 중요하면 초반에 **소셜 계정 링킹**(`POST /auth/link`,
> `references/game-auth.md` 4번)을 유도한다. 링킹해도 `userId` 는 바뀌지 않아 세이브가 그대로
> 따라온다. 플랫폼이 주는 안정적 익명키가 있다면 그것부터 쓰는 게 먼저다.

## 4. 최소 연동 예제 (순수 JS)

토큰 보관·갱신·재시도까지 웹에서 직접 처리해야 하는 부분만 담았다.

```js
const BASE_URL = 'https://api.bbbase.io';
const PROJECT_ID = '{PROJECT_ID}';
const API_KEY = '{API_KEY}';   // 클라 임베드 = 공개 취급. 소스 커밋은 하지 않는다

const store = {
  get deviceId() {
    let id = localStorage.getItem('bb_device_id');
    if (!id) { id = crypto.randomUUID(); localStorage.setItem('bb_device_id', id); }
    return id;                                  // 앱인토스면 getAnonymousKey() 의 hash 를 대신 사용
  },
  get access()  { return localStorage.getItem('bb_access'); },
  get refresh() { return localStorage.getItem('bb_refresh'); },
  get userId()  { return localStorage.getItem('bb_user_id'); },
  save({ accessToken, refreshToken, userId }) {
    localStorage.setItem('bb_access', accessToken);
    localStorage.setItem('bb_refresh', refreshToken);
    if (userId) localStorage.setItem('bb_user_id', userId);
  },
  clear() { ['bb_access','bb_refresh','bb_user_id'].forEach(k => localStorage.removeItem(k)); },
};

async function api(path, { method = 'GET', body, auth = true } = {}) {
  const headers = { 'X-API-Key': API_KEY };
  if (body) headers['Content-Type'] = 'application/json';
  if (auth && store.access) headers['Authorization'] = `Bearer ${store.access}`;

  const res = await fetch(`${BASE_URL}/projects/${PROJECT_ID}${path}`, {
    method, headers, body: body ? JSON.stringify(body) : undefined,
  });
  const json = await res.json();
  if (!json.success) {
    const err = new Error(json.error.message);
    err.code = json.error.code;        // 분기는 항상 code 로 (message 는 사람용)
    err.details = json.error.details;
    throw err;
  }
  return json.data;
}

export async function loginGuest() {
  const data = await api('/auth/guest', {
    method: 'POST', auth: false, body: { deviceId: store.deviceId },
  });
  store.save(data);
  return data.userId;                  // userId 는 BBBase 가 발급 — 직접 만들지 말 것
}

export async function refreshToken() {
  const data = await api('/auth/refresh', {
    method: 'POST', auth: false, body: { refreshToken: store.refresh },
  });
  store.save(data);                    // 회전 — 반드시 새 refreshToken 으로 교체 저장
}

// 401 이면 한 번 갱신 후 재시도, 그래도 안 되면 재로그인
export async function call(path, opts) {
  try {
    return await api(path, opts);
  } catch (e) {
    if (e.code !== 'UNAUTHORIZED') throw e;
    try { await refreshToken(); } catch { store.clear(); await loginGuest(); }
    return api(path, opts);
  }
}

export const loadSave = () => call(`/entities/user/${store.userId}/record`);
export const putSave  = (data) =>
  call(`/entities/user/${store.userId}/record`, { method: 'PUT', body: { data } });
```

주의:

- 경로의 `userId` 는 **반드시 토큰의 userId** 여야 한다(`entityType='user'` 일 때 소유권 강제).
  남의 id 를 넣으면 403 `FORBIDDEN`.
- 신규 유저는 `loadSave()` 가 `RECORD_NOT_FOUND` 로 실패한다 — 에러가 아니라 "신규"로 처리.
- 인증 API 는 **IP당 분당 10회** throttle 이므로 실패 시 재시도 루프를 돌리지 않는다.
- `USER_BANNED`(403) 는 재시도·재로그인 금지 — `details.expiresAt`/`reason` 으로 안내 화면.

## 5. Godot Web / Unity WebGL 로 내보낸 경우

SDK(`addons/bbbase/`, `Assets/BBBase/`)를 그대로 쓰면 되고 코드 변경은 없다. 다만 **§1 의
origin 등록은 똑같이 필요**하다 — 네이티브에서 잘 되던 빌드가 웹으로 내보낸 순간 로그인부터
실패한다면 거의 항상 이것이다.
