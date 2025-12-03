---
title: Findit - API 연동하기
date: 2025-12-02 17:00:00 +0900
categories: [Project, Browser-Extensions]
tags: [findit, backend, api]
---
<a href="https://github.com/victorybh04/findit/tree/main" target="_blank">Findit Github 링크</a>

## 들어가며
저번 쿠팡과 지마켓 스크래핑(크롤링) 시도는 실패로 돌아갔다. 애초에 해당 쇼핑몰들에서는 `robots.txt`에서 공인된 봇 외에는 크롤링을 허용하지 않고 있었으며, 실제 서버도 비공식 접근을 차단하고 있었다. 유저에이전트를 조작하는 수준으로는 403(Forbidden) 응답을 반환하며 접근할 수 없었다.

그래서 공식 API를 제공하는 쇼핑몰을 먼저 공략해보기로 했었다. 네이버 쇼핑과 11번가가 상품 검색 API를 제공한다는 사실을 확인했고, 오늘 해당 쇼핑몰들의 API를 이용하여 키워드로 상품을 검색하여 가져오는 기능을 만들어보고자 한다.

## API 키 발급받기
### 11번가
11번가는 기본적으로 <a href="https://openapi.11st.co.kr/openapi/OpenApiFrontMain.tmall" target="_blank">11번가 OpenAPI</a>를 제공한다. 다만 대부분의 모던 서비스들이 JSON 형식으로 데이터를 제공하는 반면, 11번가는 XML 형식으로 데이터를 제공한다. `Node.js`에서 데이터를 처리하기 위해서는 XML을 JSON 형식으로 변환하여 사용하는 것이 편리해 보인다. 또한 다른 쇼핑몰들의 데이터 처리 코드의 재활용을 위해서라도 JSON으로 형식을 변환할 필요가 있다. `xml2js`라는 라이브러리를 이용하여 XML을 JSON 형식으로 간편하게 바꿀 수 있다. 

![11st](/assets/img/findit/3-11st-api-delay.png){: w="500"}
_아뿔싸_

그렇게 API를 사용하기 위한 첫 단계로 API 키를 발급받기 위해서 위 11번가 OpenAPI 홈페이지에 접속했는데, 문제가 생겼다. 11번가에서는 API 키를 발급받기 위해서 계정을 '개인셀러'로 전환해야 할 필요가 있다. 내 기존 계정은 소셜 로그인을 이용하고 있었는데, 소셜 로그인 계정은 개인셀러로 전환할 수 없었다. 그래서 새로운 자체 계정을 만들었더니, 이미 기존의 소셜 계정에 본인인증이 되어 있어 이용할 수 없었다. 기존의 소셜 계정을 탈퇴하고 새로운 계정에서 본인인증을 시도하니, 일반회원(구매회원)의 경우 11번가 계정 탈퇴 후 30일간 재인증이 불가하다고 한다... 결론은 30일간 11번가의 API키를 발급받을 수 없게 되어버렸다. 

억울하지만 1개월 후에 API를 연동하기로 하고, 네이버 쇼핑 API 기능 구현으로 넘어갔다.

### 네이버 쇼핑

| **요청 URL** | **반환 형식** |
| - | - |
| `https://openapi.naver.com/v1/search/shop.xml` | XML |
| `https://openapi.naver.com/v1/search/shop.json` | JSON |

네이버 쇼핑 API는 위와 같이 요청하는 URL에 따라 XML과 JSON 형식 모두 데이터를 제공한다.

![naver_api](/assets/img/findit/3-naver-api-registration.png){: w="500"}
_네이버 쇼핑 API 등록 과정_

<a href="https://developers.naver.com/main/" target="_blank">네이버 개발자 센터</a>에서 `Application - 애플리케이션 등록` 탭으로 가면 API 이용을 위한 애플리케이션 등록 신청이 가능하다. 위 사진처럼 사용 API는 검색, 환경은 WEB 환경을 추가하여 등록하였다. 클라이언트 아이디와 클라이언트 시크릿 키를 발급받을 수 있었다.


## `server.js` 구현
```js
try {
    const naverApiUrl = `https://openapi.naver.com/v1/search/shop.json?query=${encodedSearchQuery}&display=1&exclude=used:rental`;   
    // 일단 최상단 1개만 가져옴, 중고와 렌탈상품 제외
    // TODO: 검색 쿼리 최적화 (쓸모없는 부분 삭제, 브랜드 및 모델명 강조 검색)

    const naverResponse = await axios.get(naverApiUrl, {
        headers: {
            'X-Naver-Client-Id': NAVER_CLIENT_ID,
            'X-Naver-Client-Secret': NAVER_CLIENT_SECRET
        },
        timeout: 5000
    });
    const naverResData = naverResponse.data;
    const productResult = [];
    productResult.push(naverResData);

    console.log(naverResData);
}
```
다음과 같이 백엔드 서버 코드를 구현했다. `naverApiUrl` 변수에 네이버 쇼핑 API 요청 Url, 쿼리(검색어), 그리고 검색 설정 몇 가지를 넣었다. 헤더에는 네이버 쇼핑 API 요청 양식에 맞게 클라이언트 아이디와 클라이언트 시크릿 키를 넣었다. get 요청에 대한 응답을 `naverResponse`에 담아, `data` 프로퍼티 값을 `naverResData`에 담았다. `productResult` 배열은 이후 다른 쇼핑몰들의 응답까지 모아 한번에 담아둘 변수이다.

![naver_api_result](/assets/img/findit/3-naver-api-test.png){: w="500"}
_네이버 쇼핑 API로 상품검색 요청을 보낸 결과_

결과는 위 사진과 같이, 요청에 성공하여 상품 정보를 받아올 수 있었다! `total`과 `start`, `display`와 같이 검색 정보에 대한 프로퍼티가 몇 개 있고, 실제 상품 정보는 `items` 프로퍼티의 배열에 존재하였다. 이 데이터에서 필요한 데이터를 몇가지를 뽑아 `res.json`을 이용해 팝업으로 보내면 실제 브라우저 익스텐션 팝업에 검색결과가 뜰 것이다. 여기까지 하면 핵심 기능인 '팝업에서 요청을 보내면, 백엔드 서버가 쇼핑몰에 상품 검색을 요청하여, 데이터를 팝업에 보내 목록을 띄운다.'가 구현되는 것이다.


```js
const naverResData = naverResponse.data.items[0];  
const productResult = [];

productResult.push({
    storeName: ['네이버쇼핑', 'navershopping'],
    title: naverResData.title,
    price: parseInt(naverResData.lprice),
    url: naverResData.link,
    shippingFeeText: '🛒 링크에서 직접 확인'   // 네이버 쇼핑 API는 배송비 정보 X
});

console.log(naverResData);
// TODO: 최적의 일치상품 하나만 찾아내는 알고리즘(로직) 필요
res.json({ results: productResult });
```
`naverResData`에 요청 결과의 `data.items` 프로퍼티를 바로 담는 것으로 수정했다. `productResult` 배열에 `naverResData`의 프로퍼티들을 가져와 객체 리터럴로 데이터를 집어넣었다. 아쉽게도 네이버 쇼핑 API의 경우 배송비 데이터를 반환하지 않아 팝업으로 보낼 수 없었다. 해당 페이지만 스크래핑하여 배송비 데이터를 넣는 방식으로 구현할 수도 있었겠지만, 일단은 MVP 구현에만 집중하여 이대로 두기로 한다. `res.json({ results: productResult });`을 통해 팝업으로 검색결과를 반환해주면 다음 사진처럼 브라우저 익스텐션 팝업에 목록이 뜨게 된다.

![popup](/assets/img/findit/3-popup.png){: w="500"}
_야호_

## 코드 개선
일단 API를 통해 정보를 받아오는 기능 구현 자체는 완성되었다. 하지만 어떤 프로젝트건, 기능구현에서 끝나는 경우는 없다. 기능구현의 완료는 오히려 새로운 시작에 가깝기 마련이다. 코드 최적화, 예외 처리, 버그 수정, 리팩터링 등 수많은 후속 작업들이 필요하다. 이번 기능의 구현과정에서 크게 네 가지의 개선점을 찾았다. 

### API 키의 보안 조치
멍청하게도, 발급받은 네이버 쇼핑 API 키(클라이언트 아이디와 시크릿)을 그대로 `server.js` 파일에 변수로 저장했고, 해당 키가 포함된 소스코드를 그대로 Github의 공개 리포지토리에 push 해버렸다... Github에 올라간 소스코드에서 키 부분을 삭제하고, 변경 히스토리도 삭제함으로써 다시 웹상에서 숨길 수는 있겠지만, 이미 공개된 공간에 올라왔었기에 키가 노출되었다고 보는 것이 맞다. 따라서 아예 재발급을 받는 것이 맞다고 판단하였다. 

찾아보니 이러한 민감한 정보(API 키, SSH 접속용 RSA 키, 데이터베이스 접근 비밀번호 등)을 관리하기 위해서 `.env`라는 파일을 만들어 사용한다고 한다. dotenv라고 불리는 이 방법론은 환경변수를 담는 파일을 만들고 커밋되지 않도록 하는 방식으로 이러한 민감한 정보를 보호한다. 

npm을 이용하여 `dotenv` 라이브러리를 설치했고, `server.js`가 있는 폴더에 `.env` 파일을 생성하였다. `.env`파일에는 새로 발급한 네이버 쇼핑 API 키들을 넣어 실제 소스코드에서 분리하였다. 
```js
require('dotenv').config();

const NAVER_CLIENT_ID = process.env.NAVER_CLIENT_ID;
const NAVER_CLIENT_SECRET = process.env.NAVER_CLIENT_SECRET;
```
`dotenv` 라이브러리를 불러오고, `.env`에서 각 키들을 가져오도록 설정했다. 그리고 `.gitignore` 파일에 `.env`을 등록하여 앞으로 커밋되지 않도록 설정하였다.

### 상품명의 `<b></b>`태그 제거 (데이터 정제)
네이버 쇼핑 API로 받아온 json 데이터의 `items` 프로퍼티에서 상품명을 의미하는 `title`엔 핵심 상품명을 강조하기 위해 `<b></b>` 태그가 섞여있다. ex) `엘지생활건강 <b>홈스타 바르는 곰팡이 싹</b> 120ml, 4개`

당장은 `title` 프로퍼티를 팝업이나 백엔드 서버에서 사용하지는 않지만, 앞으로 일치상품을 찾는 알고리즘을 구현할 때에는 이러한 태그들이 제거되어야 할 것이다. 

```js
productResult.push({
    storeName: ['네이버쇼핑', 'navershopping'],
    title: naverResData.title.replace(/<[^>]*>?/g, ''), // HTML 태그 제거
    price: parseInt(naverResData.lprice),
    url: naverResData.link,
    shippingFeeText: '🛒 링크에서 직접 확인'   // 네이버 쇼핑 API는 배송비 정보 X
});
```
이 태그를 제거하기 위해 `replace()` 메서드와 정규식(`RegExp`)을 사용하기로 했다. 단순히 `<b>`, `</b>`를 찾아 없앨 수도 있지만, 혹시모를 경우 다른 태그가 `title`에 들어가 있을지도 모른다. 따라서 HTML의 태그 형식은 전부 걸러내는 정규식을 작성해 위와 같이 수정했다.

### 검색결과가 없을 시 예외 처리
현재 구현 단계에서는 최적의 일치 상품을 찾는 알고리즘 없이, 임시로 첫 페이지 첫번째 상품을 가져와 반환한다.
```js
const naverResData = naverResponse.data.items[0];
```
그래서 위와 같은 형식으로 하드코딩해 두었는데, 만약 네이버쇼핑 API에서 검색어에 해당하는 상품을 찾을 수 없어 빈 `items`를 보냈다면 `naverResData`는 `undefined` 값이 될 것이다. 이 상태에서 `productResult` 배열에 `naverResData.프로퍼티`들을 접근한다면 `undefined.프로퍼티`를 참조하게 되어 에러가 발생할 것이다. 에러 때문에 서버가 멈추는 것을 방지하기 위해, 다음과 같이 `items`가 빈 경우에 대한 예외 처리 코드를 작성했다.
```js
const productResult = [];

if (naverResponse.data.items && naverResponse.data.items.length > 0){
    const naverResData = naverResponse.data.items[0];
    
    productResult.push({
        storeName: ['네이버쇼핑', 'navershopping'],
        title: naverResData.title.replace(/<[^>]*>?/g, ''),
        price: parseInt(naverResData.lprice),
        url: naverResData.link,
        shippingFeeText: '🛒 링크에서 직접 확인'   // 네이버 쇼핑 API는 배송비 정보 X
    });
}
```

### 파라미터 분리
```js
const naverApiUrl = `https://openapi.naver.com/v1/search/shop.json?query=${encodedSearchQuery}&display=1&exclude=used:rental`;
```
기존에는 위 코드와 같이 `naverApiUrl`에 요청 URL 뿐만 아니라, 검색할 쿼리와 검색 옵션 파라미터까지 저장해서 get 요청을 보냈다. 이 코드는 보기 불편하며 유지보수성이 떨어진다. (예를 들어 나중에 중고상품도 포함하여 검색하는 기능을 추가한다면, 쿼리문에서 `used`를 빼야 하는데, 이는 `naverApiUrl` 문자열을 수정해야 하기에 복잡해진다.) 이 백엔드 서버에서는 `Axios`를 사용하고 있는데, `Axios`의 `params` 옵션을 이용하면 훨씬 편리하게 쿼리와 파라미터를 지정해 요청을 보낼 수 있다. 그래서 다음과 같이 코드를 작성했다.

```js
const naverApiUrl = `https://openapi.naver.com/v1/search/shop.json`;

const naverResponse = await axios.get(naverApiUrl, {
    params: {
        query: searchQuery,     // params 문법에서는 자동으로 query 인코딩
        display: 1,             // 일단 최상단 1개 상품만 가져옴
        exclude: 'used:rental'  // 중고 및 렌탈상품 제외
    },
    headers: {
        'X-Naver-Client-Id': NAVER_CLIENT_ID,
        'X-Naver-Client-Secret': NAVER_CLIENT_SECRET
    },
    timeout: 5000
});
```
이렇게 작성하면 `Axios`의 `param` 옵션은 자동으로 쿼리를 인코딩해주기 때문에 검색어 쿼리를 따로 인코딩하여 넣을 필요가 없고, 이후 알고리즘이 완성되면 `display` 파라미터를 수정하여 상품검색의 범위를 편하게 변경할 수 있다. 또한 중고와 렌탈 상품의 포함여부를 편리하게 조정할 수 있을 것이다. 


## 마무리하며
API의 정의는 다음과 같다.
> API는 'Application Programming Interface'의 약자로, 서로 다른 소프트웨어 애플리케이션들이 소통하고 데이터를 교환할 수 있도록 하는 규칙과 정의의 집합입니다.

개발을 공부하며, API라는 단어 자체는 여러 번 들어봤다. 하지만 위와 같은 정의만으로는 'API가 다른 애플리케이션 간에 소통하는 방식인건 알겠는데, 그래서 그게 어떻게 생긴건데'라는 생각이 들며 그 개념이 이해가 잘 되지 않았다. 사실 API는 추상적인 규약에 가깝고, API의 개발자와 기능, 목적에 따라 구현된 방식이 천차만별이다. 때문에 API를 실제로 사용해보지 않고서는 어떻게 구성되어 있고, 어떤 식으로 작동하는 것인지 이해하기 쉽지 않았다. 

이번 포스트에서 Findit 백엔드 서버는 쇼핑몰 OpenAPI 서버에 HTTP 요청을 통해 상품 검색결과를 받아온다. 각 쇼핑몰의 OpenAPI 키를 발급받고 이용해보며 API의 개념과 그 사용 방식을 확실히 이해하게 되었다. 개념 공부만 100번 하는 것 보다 직접 무언가를 만들어보며 사용해보는 것이 훨씬 도움이 된다는 사실을 깨닫게 된 순간이었다.

