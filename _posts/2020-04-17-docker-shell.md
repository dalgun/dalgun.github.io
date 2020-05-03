---
layout: post
title: "Docker registry remote delete"
subtitle: "docker image 원격 삭제 shell script"
author: "Dalgun"
category: "Docker"
tag: [docker, shell script, rest, docker-registry]
comments: true
---

![thumb](https://mblogthumb-phinf.pstatic.net/20160807_7/alice_k106_1470553952857vehWE_PNG/private-containers.png?type=w800)

# 저장용 

Private Docker Registry 에 SNAPSHOT 버전의 경우 깔끔하게 이미지를 올리기 위해서 항상 이미지를 삭제하고 올렸는데,    
너무 귀찮아서 쉘스크립트를 작성해보았습니다ㅋㅋㅋ 혹시 도움 되실분들이 있으실까 해서 올려봅니다

```bash
#!/bin/bash
registry='{docker registry url}'
name=$1
auth='-u username:password'
tag=$2

## delete images and/or tags
echo "[${registry} 로컬 이미지 삭제]"
OUTPUT=$(docker image inspect ${registry}/${name}:${tag} >/dev/null 2>&1 && echo yes || echo no)
echo "[NAME]:${name}"
echo "[tag]:${tag}"

if [[ "$OUTPUT" == "yes" ]]
then
    echo -n "docker rmi ${registry}/${name}:${tag}"
    docker rmi ${registry}/${name}:${tag}
fi

echo "[${registry} 리모트 이미지 삭제]"
SHA=$(curl $auth -sI -k -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://${registry}/v2/${name}/manifests/${tag}" | tr -d '\r' | sed -En 's/^docker-content-digest: (.*)/\1/p')

echo "SHA:${SHA}"
curl $auth -X DELETE -sI -k "https://${registry}/v2/${name}/manifests/${SHA}"
```


실행할때는

```bash
$> chmod 755 docker-rmi.sh
$> ./docker-rmi.sh redis latest
```

짜잔

