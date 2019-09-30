# APIGatewayAuthDemo

## 概要

API GatewayのリクエストにCognitoによる認証を組み込んだデモ

## 構成



## デプロイ

```sh
aws cloudformation create-stack \
    --stack-name api-gateway-auth-demo \
    --template-body file://template.yml
```

## 使い方

### ユーザ作成

ユーザプールIDを取得

```sh
USER_POOL_ID=$(aws cloudformation describe-stacks \
    --stack-name simple-authenticate \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolId`].OutputValue' \
    --output text)
echo ${USER_POOL_ID}
  # ap-northeast-1_ydrdhvOdG
```

ユーザ名は `ot-nemoto@example.com` / 初期パスワードは `Passw0rd?`

```sh
aws cognito-idp admin-create-user \
    --user-pool-id ${USER_POOL_ID} \
     --username ot-nemoto@example.com \
    --temporary-password Passw0rd?
```

### 初回認証

ユーザプールのクライアントIDを取得

```sh
USER_POOL_CLIENT_ID=$(aws cloudformation describe-stacks \
    --stack-name simple-authenticate \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolClientId`].OutputValue' \
    --output text)
echo ${USER_POOL_CLIENT_ID}
  # 4rt9kn7rd5c7av4686tec5p21k
```

認証

```sh
aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --client-id ${USER_POOL_CLIENT_ID} \
     --auth-parameters USERNAME=ot-nemoto@example.com,PASSWORD=Passw0rd?
  # {
  #     "ChallengeName": "NEW_PASSWORD_REQUIRED",
  #     "ChallengeParameters": {
  #         "USER_ID_FOR_SRP": "d0b2ec76-ac3b-47b8-ad14-3aa7fac7c102",
  #         "requiredAttributes": "[]",
  #         "userAttributes": "{\"email\":\"ot-nemoto@example.com\"}"
  #     },
  #     "Session": "1a2lI3uS03...JzBV0cyng4"
  # }
```

### パスワード変更

ユーザ作成時には、ユーザのステータスが `FORCE_CHANGE_PASSWORD` であるため、トークンIDを取得できない
Sessionを利用し、初期パスワードを更新する

以下では、パスワードを `qwerT1234%` に変更

```sh
aws cognito-idp respond-to-auth-challenge \
    --client-id ${USER_POOL_CLIENT_ID} \
    --challenge-name NEW_PASSWORD_REQUIRED \
    --challenge-responses USERNAME=ot-nemoto@example.com,NEW_PASSWORD=qwerT1234% \
    --session "1a2lI3uS03...JzBV0cyng4"
  # {
  #     "AuthenticationResult": {
  #         "ExpiresIn": 3600,
  #         "IdToken": "eyJraWQiOi...QJuwtGQ_dQ",
  #         "RefreshToken": "eyJjdHkiOi...4fhSK18PfA",
  #         "TokenType": "Bearer",
  #         "AccessToken": "eyJraWQiOi...5u3c0Xibsg"
  #     },
  #     "ChallengeParameters": {}
  # }
```

### トークンID取得

```sh
```

### APIを実行

トークンID未設定でAPIを実行

```sh
curl -s -XGET ${INVOKE_URL} | jq
  # {
  #   "message": "Unauthorized"
  # }
```

トークンIDを取得

```sh
TOKEN=$(aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --client-id ${USER_POOL_CLIENT_ID} \
     --auth-parameters USERNAME=ot-nemoto@example.com,PASSWORD=qwerT1234% \
     --query 'AuthenticationResult.IdToken' \
     --output text)
echo ${TOKEN}
  # eyJraWQiOi...lJcHl_o_kg
```

APIを実行

```sh
curl -s -XGET ${INVOKE_URL} -H "Authentication:${TOKEN}" | jq
  # {
  #   "message": "Hello"
  # }
```
