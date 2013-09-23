---
layout: post
title: Procastacoding
---

Since I should be applying for jobs and doing other boring paperwork oriented
tidbits at the moment I thought I'd see if I could make a rasteriser fit on a
business card a la [Business Card Ray-tracer](http://fabiensanglard.net/rayTracing_back_of_business_card/index.php).

I ripped the code from [@rygorous](twitter.com/rygorous)' fantastic [software
occlusion culling series](http://fgiesen.wordpress.com/2013/02/17/optimizing-sw-occlusion-culling-index/)
but I haven't made it actually do anything interesting. If I cut back the
performance stuff and have a bit of a dip at shortening the maths and triangle
specification I should be able to make it smaller but the real issue I had was
coming up with something worth rasterising. Maybe reading a simple mesh format
from stdin would be the best option.

Anyway here's the broken code (doesn't have a consistent fill rule, no
sub-pixel precision, doesn't actually test the depth so it's basically the
painters algorithm, fails to render anything remotely interesting)

```c
#include<stdio.h>  // ./rast > test.ppm
#include<stdlib.h>
#include<string.h>
#define W 800
#define H 600
#define X(a,b) (a)>(b)?(a):(b)
#define X3(a,b,c) X(X(a,b),(c))
#define N(a,b) (a)<(b)?(a):(b)
#define N3(a,b,c) N(N(a,b),(c))
typedef float f;typedef int i;char b[W*H]={};struct F{f x,y,z;};struct I
{i x,y,z;I&operator+=(I&o){x+=o.x;y+=o.y;z+=o.z;return*this;}};i o(I&a,I
&b,I&c){return (b.x-a.x)*(c.y-a.y)-(b.y-a.y)*(c.x-a.x);}i main(){I t[]={
{10,10,0},{115,10,128},{10,115,0},{10,120,0},{125,15,255},{125,125,0}};
for(i j=0;j<sizeof(t)/sizeof(t[0]);j+=3){I v0=t[j],v1=t[j+1],v2=t[j+2];I
n={X(N3(v0.x,v1.x,v2.x),0),X(N3(v0.y,v1.y,v2.y),0)},x={N(X3(v0.x,v1.x,
v2.x),W-1),N(X3(v0.y,v1.y,v2.y),H-1)},A={v1.y-v2.y,v2.y-v0.y,v0.y-v1.y},
B={v2.x-v1.x,v0.x-v2.x,v1.x-v0.x},p=n,r={o(v1,v2,p),o(v2,v0,p),o(v0,v1,p
)};f a=1.0f/(r.x+r.y+r.z);F Z={v0.z,(v1.z-v0.z)*a,(v2.z-v0.z)*a};for(;
p.y<=x.y;p.y++){I w=r;for(p.x=n.x;p.x<=x.x;p.x++){i z=Z.x*w.x+Z.y*w.y+
Z.z*w.z;if((w.x|w.y|w.z)>=0) b[p.y*W+p.x]=z;w+=A;}r+=B;}}printf("P5 %d "
"%d 255 ",W,H);for(i y=H;y--;)for(i x=W;x--;)printf("%c",b[y*W+x]);}
```

But for now I should probably get back to applying for jobs.
