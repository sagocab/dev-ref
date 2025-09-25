# Vue + Axios 비동기 요청 후 Redirect 문제 해결

Vue 프론트엔드 화면에서 Axios를 이용해 서버와 통신할 때,  
**POST → GET → Redirect** 순서를 지켜야 하는데 비동기 처리 문제로  
중간에 에러 화면이 먼저 뜨고 이후 redirect 화면으로 이동하는 문제가 발생할 수 있습니다.

---

## 문제 원인
- Axios는 기본적으로 **비동기** 동작을 하기 때문에 순서가 보장되지 않음
- Redirect URL을 받기 전에 잘못된 라우팅이 발생 → 에러 화면 표시 → 그 후 redirect 수행

---

## 해결 방법

### 1. `async/await` 사용
```javascript
methods: {
  async handleRedirect() {
    try {
      // 1. POST 요청
      const postResponse = await axios.post('/api/check', { someValue: this.someValue });

      if (postResponse.data && postResponse.data.condition === true) {
        // 2. GET 요청
        const getResponse = await axios.get(`/api/redirect?param=${postResponse.data.param}`);
        
        if (getResponse.data && getResponse.data.redirectUrl) {
          // 3. redirect
          window.location.href = getResponse.data.redirectUrl;
        }
      }
    } catch (error) {
      console.error("요청 처리 중 오류 발생", error);
      // 필요시 사용자 친화적인 에러 화면 처리
    }
  }
}
```

---

### 2. `Promise.then` 체인 방식
```javascript
axios.post('/api/check', { someValue: this.someValue })
  .then(postRes => {
    if (postRes.data.condition) {
      return axios.get(`/api/redirect?param=${postRes.data.param}`);
    }
  })
  .then(getRes => {
    if (getRes?.data?.redirectUrl) {
      window.location.href = getRes.data.redirectUrl;
    }
  })
  .catch(err => {
    console.error("에러 발생", err);
  });
```

---

### 3. 에러 화면 깜빡임 방지
Redirect URL을 받기 전까지는 라우터 이동을 막고,  
Redirect URL을 받은 후에만 이동을 실행해야 함.

```javascript
data() {
  return {
    isLoading: false
  }
},
methods: {
  async handleRedirect() {
    this.isLoading = true; // 로딩 상태 on
    try {
      const postRes = await axios.post('/api/check', { someValue: this.someValue });
      if (postRes.data.condition) {
        const getRes = await axios.get(`/api/redirect?param=${postRes.data.param}`);
        window.location.href = getRes.data.redirectUrl;
      }
    } catch (e) {
      console.error("에러 발생", e);
      this.$router.replace('/error'); // 필요시 에러 페이지 이동
    } finally {
      this.isLoading = false; // 로딩 상태 off
    }
  }
}
```

---

## 정리
- 반드시 `async/await` 혹은 `then` 체인으로 요청 순서를 보장해야 함
- Redirect URL을 받기 전까지는 라우터 이동을 막아야 에러 화면이 깜빡이지 않음
- Vue Router 사용 여부에 따라 `this.$router.push()` 또는 `window.location.href` 방식을 선택
