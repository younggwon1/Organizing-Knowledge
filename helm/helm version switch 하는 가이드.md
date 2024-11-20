# helm version switch 하는 가이드
v2 < - > v3기존 argocd는 helm v2로 설치가 되었기 때문에 helm v3일 경우 helm version switch가 필요하다.

1. helm-switcher install
`brew install tokiwong/tap/helm-switcher`
2. helm switch
  `helmswitch -b /opt/homebrew/bin/helm`