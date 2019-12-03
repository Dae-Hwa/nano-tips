## 문제점/사용배경

### Jackson 라이브러리가 제대로 동작하지 않음

- @RequestBody 어노테이션이 제대로 작동하지 않음
- 잭슨 라이브러리가 제대로 실행되지 않는다고 추정
- 메이븐 레포지토리 사용 불가능한 상태
- 디펜던시 추가가 아닌, 라이브러리 추가로 사용해야 하는 상황
- 의존관계 등록이 되지 않았다고 판단
    - 이에 따라 메세지 컨버터가 제대로 작동이 되지 않는다고 판단
    - 잭슨은 메세지를 받는 순간 해당 컨텐츠 타입을 확인하여 정해진 규칙으로 파싱해주는 라이브러리
- 메세지 컨버터 등록
    - [https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/http/converter/json/MappingJackson2HttpMessageConverter.html](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/http/converter/json/MappingJackson2HttpMessageConverter.html)
    - [https://devjackie.tistory.com/entry/spring-32-이상에서-사용할-수-있는-messageConverter-방법](https://devjackie.tistory.com/entry/spring-32-%EC%9D%B4%EC%83%81%EC%97%90%EC%84%9C-%EC%82%AC%EC%9A%A9%ED%95%A0-%EC%88%98-%EC%9E%88%EB%8A%94-messageConverter-%EB%B0%A9%EB%B2%95)
    - [https://xens.tistory.com/entry/test](https://xens.tistory.com/entry/test)

<br/>

### Jackson 사용 시 415에러 발생(unsupported media type)

- 잭슨 라이브러리 메세지 등록 후 발생

    → 메세지 컨버터는 제대로 연결되었다 판단

- 클라이언트에서 요청 시 헤더에 Accept 추가
    - 이 방법으로 해결되지 않음

<br/>

### javax.json 사용

- 현재 @RequestBody 정상 작동 불가능
- 메이븐 레포지토리 사용 불가
- 외부 라이브러리 추가 적용이 힘든 상황
- 따라서 @RequestParam 어노테이션 사용
- 1.7 EE 버전부터 기본 내장
- 튜토리얼-[https://www.baeldung.com/jee7-json](https://www.baeldung.com/jee7-json)

<br/>

## 클라이언트

- 데이터 전송 시

  ```js
  data : {
    keyForMap : JSON.strigify({
      keyForJSON : value[or obj],
    }),
  }
  ```

  - `Stringify` 를 해주지 않으면 클라이언트에서 배열로 변환하여 보내주는데, <br/>
  라이브러리가 정상 작동하지 않으므로 이를 인식하지 못하고 단순 텍스트 형태가 된다.
    
    ```js
    data :{
      key1 : {
        key2 : value
      }
    }
    ```
    위의 데이터 전송 시

    `data[key1][key2] = value` 와 같은 `String` 값이 전달된다. 
    
    잘 가공하면 되겠으나, 너무 번거로운 작업이다.

<br/>

## 서버

### 데이터 전 처리

`JSON.Stringify` 로 처리를 하여 전송 시 다음과 같은 내용이 전달된다.

```java
keyForMap ="{&quot;keyForJson&quot;:&quot;value&quot;}"
```

따라서 적절히 파싱해준다.

```java
parsedJsonStr = requestParam.get("keyForMap").replaceAll("&quot;","\"");
```

<br/>

### JSON 변환 준비

`Json.createReader()` 메소드를 이용하여 `JsonReader` 타입의 인스턴스를 생성한다.

```java
JsonReader jsonReader = Json.createReader(new StringReader(parsedJsonStr));
```

> 사용이 끝난 뒤, `jsonReader.close()` 를 해준다.

<br/>

### 타입 체크

javax.json 은 다음과 같은 enum 타입으로 타입을 확인한다

- [https://docs.oracle.com/javaee/7/api/javax/json/JsonValue.ValueType.html](https://docs.oracle.com/javaee/7/api/javax/json/JsonValue.ValueType.html)

또한 다음과 같은 인터페이스를 제공하는데 위의 타입과 동일한 타입인 인터페이스를 맞춰 사용해주면 된다.

- [https://docs.oracle.com/javaee/7/api/javax/json/package-summary.html#package.description](https://docs.oracle.com/javaee/7/api/javax/json/package-summary.html#package.description)

> 현재 `keyForMap` 을 이용하여 불러온 스트링 값이 `{}` 로 묶여있다. 즉 객체이기 때문에, OBJECT 타입을 선택한다. 만약 `[]` 로 묶여있다면 ARRAY 타입을 선택하여 작업한다.

```java
JsonObj jsonObj = jsonReader.readObject();
// readObject(), readArray(), read() 이용 가능
// read()는 OBJECT 혹은 ARRAY 타입에 따라 리턴 
```

<br/>

하지만, 타입 체크를 하는데에 약간의 문제점이 있다.

`JsonReader` 만 이용하여 타입 체크가 불가능하다.

`JsonObject` 혹은 `JsonArray` 등의 클래스에는 `getValueType()` 메소드가 존재하지만, `JsonReader` 클래스에는 존재하지 않는다. 따라서 타입체크를 해야하는 상황이라면 클래스를 추적해야 한다.

> 혹시 다른 방법이 있다면 코멘트 부탁드립니다.

```java
Object obj = jsonReader.read();
String InterfaceName = obj.getClass().getInterfaces[0].getName();

if(InterfaceName.equals("javax.json.JsonObject")){

} else if(InterfaceName.equals("javax.json.JsonArray")){

}
```

이후 부터는 key 값을 안다면 쉽게 체크 가능하다.

<br/>

### 원하는 값 탐색

`JsonArray` 와 `JsonObject` 는 `collection` 을 구현했기때문에 쉽게 원하는 값을 탐색할 수 있다.

```java
JsonArray getJsonArray(JsonObj jsonObj){
  // JsonObject는 Map을 사용
  for(String key : jsonObj.keySet()){
    if(jsonObj.get(key).getValueType().equals(JsonValue.ValueType.Array)){
      return jsonObj.getJsonArray(key);
    }	
  }
  return null;
}
```

```java 
JsonArray jsonArray = getJsonArray(jsonObj);
// jsonArray는 List를 사용
for(int index = 0; index < jsonArray.size(); index++){
  ...;
}
```

당연히 위 방법이 아닌, 이터레이터를 이용하여 찾을 수도 있다.

<br/>

주의할 점은 `JsonString` 타입의 객체를 반환 한 경우 `toString()` 메소드를 이용하여 문자열을 추출해야 한다.

또한, 이 경우 따옴표가 붙어있는 상태로 문자열 반환이 되기때문에 파싱을 한 뒤 사용해야 한다.

<br/>

## 좋았던 점

- EE 1.7 이후 버전에서는 내장되어 있기 때문에 제한적 상황에서 접근성이 비교적 좋을 듯하다.
- 또한, 스프링 메세지 컨버터가 읽어오는게 아니기 때문에 파싱 시 커스터마이징이 필요한 경우 오히려 덜 복잡할 수 있을 듯 하다.

## 아쉬운 점

- 프레임워크에 종속되는 경향이 덜 한 것이 장점이라 생각되는데, 이미 그런 라이브러리는 많다.
- VO매핑이 힘들다.
- 컬렉션을 구현했지만, 해당 타입으로 추가 작업을 하기 힘들다.

    ex) 마이바티스의 인자로 받아오는 등
