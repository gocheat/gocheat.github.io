---
layout: single
title:  "플러터(Flutter) JSON 직렬화를 통한 API 요청하기"
date: 2021-03-01
tags: [flutter, api, serializable]
categories: flutter
---

플러터에서 Restful API 요청을 위해서는 JSON 직렬화 방식을 선정하는것이 중요한데, 크게 2가지 방식을 선택할 수 있습니다.

1. 일반 직렬화
2. 코드 생성을 이용한 자동화된 직렬화

이번 시간은 2번째 코드 생성을 통한 직렬화 방식에 대해 설명하도록 하겠습니다.

### 1. json_serializable 설정

우선 자동 코드 생성을 위해 라이브러리 의존성 1개와 라이브러리 개발 의존성 2개를 설치 합니다.

```yaml
// pubspec.yaml 파일에 추가

dependencies:
  ...
  json_annotation: ^2.0.0

dev_dependencies:
  ...
  build_runner: ^1.0.0
  json_serializable: ^2.0.0
```
프로젝트 루트에서 `flutter pub get` 명령어를 실행하여 의존성을 사용할수 있도록 합니다.

### 2. json_serializable를 통한 요청,응답 직렬화 클래스 생성하기

우선 API 요청 코드를 짜기 위해서는 `Request`,`Response`에 대해 직렬화가 필요한 클래스를 생성해줍니다.

예를 들어서 로그인 API 를 개발한다고 가정해보겠습니다. 그렇다면 로그인 API 요청할 직렬화 클래스를 아래와 같이 작성합니다.

```dart
import 'package:json_annotation/json_annotation.dart';
/// 이 구문은 "LoginRequest","LoginResponse" 클래스가 생성된 파일의 private 멤버들을
/// 접근할 수 있도록 해줍니다. 여기에는 *.g.dart 형식이 들어갑니다.
/// * 에는 소스 파일의 이름이 들어갑니다.
part 'auth.g.dart';
        
// 이 클래스가 JSON 직렬화 로직이 만들어져야 한다고 알려주는 어노테이션
@JsonSerializable()
class LoginRequest {
  LoginRequest({this.type, this.accessToken});

  final int type;

  // JsonKey 애노테이션을 통해 API에서 snake_case로 반환해줍니다.
  @JsonKey(name: 'access_token')
  final String accessToken;

  /// map에서 새로운 LoginRequest 인스턴스를 생성하기 위해 필요한 팩토리 생성자입니다.
  /// 생성된 `_$LoginRequestFromJson()` 생성자에게 map을 전달해줍니다.
  factory LoginRequest.fromJson(Map<String, dynamic> json) => _$LoginRequestFromJson(json);

  /// `toJson`은 클래스가 JSON 인코딩의 지원을 선언하는 규칙입니다.
  /// 이의 구현은 생성된 private 헬퍼 메서드 `_$LoginRequestToJson`을 단순히 호출합니다.
  Map<String, dynamic> toJson() => _$LoginRequestToJson(this);
}

@JsonSerializable()
class LoginResponse{
  final String token;

  LoginResponse({this.token});

  factory LoginResponse.fromJson(Map<String, dynamic> json) => _$LoginResponseFromJson(json);
  Map<String, dynamic> toJson() => _$LoginResponseToJson(this);
}

```

위의 코드 작성을 마치면 툴 내에서 존재하지 않는 함수 및 파일이라고 에러가 나올것입니다. 때문에 아래 코드 생성을 마쳐야만 사용 가능한 상태가 됩니다.

### 3. 직렬화 코드 자동 생성하기

`flutter pub run build_runner build` 명령어를 루트에서 입력하게 되면 JSON 직렬화 코드를 생성할 수 있습니다. 그러나 매번 수정에 따른 일회성으로 하는 코드 생성은 시간이 오래 걸리기때문에 
`flutter pub run build_runner watch` 명령어를 통해 프로젝트 파일들의 변화를 지켜보고 자동으로 코드를 생성하게 해줄 수 있습니다.

위의 명령을 처리하고 나면 `auth.g.dart` 라는 파일이 생긴것을 확인 할수 있습니다. 아래는 자동으로 생성된 코드 입니다.

```dart
// auth.g.dart 
// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'auth.dart';

// **************************************************************************
// JsonSerializableGenerator
// **************************************************************************

LoginRequest _$LoginRequestFromJson(Map<String, dynamic> json) {
  return LoginRequest(
    type: json['type'] as int,
    accessToken: json['accessToken'] as String,
  );
}

Map<String, dynamic> _$LoginRequestToJson(LoginRequest instance) =>
    <String, dynamic>{
      'type': instance.type,
      'accessToken': instance.accessToken,
    };

LoginResponse _$LoginResponseFromJson(Map<String, dynamic> json) {
  return LoginResponse(
    token: json['token'] as String,
  );
}

Map<String, dynamic> _$LoginResponseToJson(LoginResponse instance) =>
    <String, dynamic>{
      'token': instance.token,
    };

```

### 4. LOGIN API 호출 최종 코드 

아래는 위의 직렬화 객채를 통해 Login API를 요청하는 예제 코드입니다. 

```dart
@JsonSerializable()
class LoginRequest {
  LoginRequest({this.type, this.accessToken});

  final int type;
  final String accessToken;

  factory LoginRequest.fromJson(Map<String, dynamic> json) => _$LoginRequestFromJson(json);
  Map<String, dynamic> toJson() => _$LoginRequestToJson(this);
}

@JsonSerializable()
class LoginResponse{
  final String token;

  LoginResponse({this.token});

  factory LoginResponse.fromJson(Map<String, dynamic> json) => _$LoginResponseFromJson(json);
  Map<String, dynamic> toJson() => _$LoginResponseToJson(this);
}

// 로그인 API
Future<LoginResponse> loginAPI(String accessToken) async {
  var request = LoginRequest(type: 1, accessToken: accessToken);
  var response = await http.post(
      'http://localhost:8080/auth/token',
      headers: <String, String>{
        'Content-Type': 'application/json; charset=UTF-8',
      },
      body: jsonEncode(request)
  );
  print(response.body.toString() + "["+ response.statusCode.toString() +"]");
  if (response.statusCode == 200) {
    var loginResponse = LoginResponse.fromJson(jsonDecode(response.body));
    return loginResponse;
  } else {
    // 응답이 200이 아닐경우 예외처리
    throw Exception('failed to oauth login');
  }
}
```

### 참고문서
- Flutter 공식 문서: <https://flutter-ko.dev/docs/development/data-and-backend/json>
