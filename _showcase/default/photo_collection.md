---
show: true
width: 4
date: 2021-09-12 00:01:00 +0800
height: 400px
images:
- src: ./assets/images/photos/Liverpool.jpg
  title: Liverpool
  desc: Liverpool, UK
- src: ./assets/images/photos/kunmingpool.jpg
  title: Kunming Pool
  desc: Xi'an, China
- src: ./assets/images/photos/nanning.jpg
  title: Nanning
  desc: Nanning, China
---

{% include widgets/carousel.html id=page.id images=page.images height=page.height %}
