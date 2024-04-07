# Amplify 이슈 및 해결

### Framework Web not supported

1. 이슈 : 2023-08-14T08:27:16.101Z [ERROR]: !!! CustomerError: Framework Web not supported
2. 해결 : aws amplify update-branch --app-id "{app-id}" --branch-name "{branch-name}" --framework 'Next.js - SSR'
   - Ex) aws amplify update-branch --app-id "d4fjkv943dvbt" --branch-name "main" --framework 'Next.js - SSR'





### Platform : WEB 이슈

1. 이슈 : amplify platform 이 web compute 가 아닌 web 일 경우 배포가 안돼는 이슈
2. 해결 : aws amplify update-app --app-id "{app-id}" --platform WEB_COMPUTE
   - Ex) aws amplify update-app --app-id "d4fjkv943dvbt" --platform WEB_COMPUTE