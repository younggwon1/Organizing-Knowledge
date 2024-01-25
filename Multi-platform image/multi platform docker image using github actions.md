하나의 Dockerfile을 기반으로 여러 플랫폼에서 실행 가능한 Docker 이미지를 생성하는 것을 의미한다.

```
- name: Build and push
uses: docker/build-push-action@v4
with:
    context: .
    push: true
    tags: user/repo:latest
    platforms: |
        linux/amd64
        linux/arm64
        linux/arm/v7
        ...
```

https://docs.docker.com/build/ci/github-actions/multi-platform/

https://github.com/marketplace/actions/build-and-publish-multi-platform-docker-images

https://thearchivelog.dev/article/building-a-multi-platform-image-with-github-actions/
