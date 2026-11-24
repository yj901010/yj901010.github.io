---
title: "로그는 다 있는데, 장애 원인은 왜 찾기 어려울까?"
date: 2026-11-24 17:21:00 +0900
categories: [Backend, Observability]
tags: [netflix, observability, knowledge-graph, incident, autorca]
---

장애가 나면 일단 로그부터 보게 된다. 그런데 에러가 난 곳을 찾았다고 원인까지 바로 알게 되는 건 아니다.

“이 서비스 직전에 뭐 배포했지? 전에 비슷한 문제 있었나?”

이걸 찾으려고 다른 도구를 열고, 예전 대화도 검색하게 된다.

넷플릭스 기술 블로그에도 이런 이야기가 있었다. 홈 화면이 느려지는 상황을 예로 들면서, 조사 시간이 흩어진 정보의 관계를 다시 찾는 데 많이 쓰인다고 설명한다. 글에 나오는 4시간은 이 설명용 사례의 시간이다.

메트릭은 Atlas에, 배포 이력은 Spinnaker에, 과거 장애를 해결한 대화는 Slack에 있었다. 게다가 같은 대상을 도구마다 다른 이름으로 부르기도 했다.

그러면 결국 아는 사람이 연결해줘야 한다.

“거기 나온 그 이름이 여기서는 이 서비스야.”

정보를 모아놨어도 이런 관계는 사람이 기억하고 있어야 했던 거다. [Netflix 원문](https://netflixtechblog.com/ontology-driven-observability-building-an-end-to-end-knowledge-graph-at-netflix-scale-421392cf5adf)

넷플릭스는 먼저 여러 도구의 기록이 같은 애플리케이션을 가리키도록 기준을 맞췄다. 그리고 담당 팀, 배포, 장애 사이의 관계를 지식 그래프로 저장했다.

원문의 예시를 간단히 옮기면 이런 식이다.

```text
api-gateway → 담당 팀 → Team Bedrock
api-gateway → 배포 지역 → us-east-1
장애 INC-5377 → 영향받은 애플리케이션 → api-gateway
```

이렇게 연결하면 “이 팀이 담당하는 애플리케이션에 어떤 장애가 있었지?”를 여러 도구에서 따로 찾아 합칠 필요가 줄어든다.

그 위에서 AutoRCA라는 사내 AI 시스템이 관련 정보를 따라가며 원인 후보를 제시한다. 후보가 틀릴 수도 있어서, 왜 그렇게 판단했는지 근거를 함께 남기고 엔지니어가 확인하도록 했다. [원문의 AutoRCA 설명](https://netflixtechblog.com/ontology-driven-observability-building-an-end-to-end-knowledge-graph-at-netflix-scale-421392cf5adf)

읽고 나니 로그를 남기는 것 다음에 생각할 일이 보였다.

로그에 서비스 이름이 있는데 배포 기록에서는 다른 이름을 쓰고 있다면, 나중에 누가 그 둘을 연결해줄까? 담당자가 자리에 없으면 또 찾아봐야 하는 걸까?

장애가 났을 때 그런 것부터 찾고 싶지는 않다. 어떤 정보끼리 연결되는지 평소에 남겨두는 것도 장애 대응을 준비하는 일이겠다는 생각이 들었다.

---

참고로 이름이 비슷한 [AutoRCA-Core](https://github.com/nik-kale/AutoRCA-Core#readme)는 Nik Kale이 공개한 별도 프로젝트다. 이 글에서 설명한 넷플릭스의 사내 AutoRCA와는 구분해서 봐야 한다.
