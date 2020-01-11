---
layout: post
title:  "Matchstick squares: solution details"
date:   2019-10-08
---

See [Matchstick Squares](matchsticks.html) for background. Below are
details on the solutions to matchstick-square problems for various
sizes.

The OEIS has a [full list](https://oeis.org/A294249/a294249.txt) of
what I believe to be the optimal answers up through 46 squares, and
some best-known ones up through 63 squares. Where one N has multiple
best solutions which are qualitatively different, I've given examples
of each type.

Legend:
- N: size of largest square
- Full: a full exhaustive search
- Diagonal-1: **all** squares must be on the main diagonal
- Diagonal-20: all squares of size 21 and over must be on the main
diagonal
- Greedy: only consider placements which add the minimum number of
matches
- <span style="background:lightgreen">Green</span> shows where one
heuristic (greedy or diagonal-20) beats the other
- <span style="background:pink">Pink</span> shows where an exhaustive
search beats the greedy and diagonal heuristics

| N	| # optimal solutions | Best full solutions | Best diagonal-1 solution | Best diagonal-20 solution | Best greedy solution |
| ---   | ---     | ---     | ---       | ---   | --- |
| 1	| 1	  | 4	    | 4		| 4	| 4   |
| 2	| 1	  | 10	    | 10	| 10	| 10  |        
| 3	| 1	  | 17	    | 18	| 17	| 17  |
| 4	| 1	  | 26	    | 26	| 26	| 26  |
| 5	| 3	  | 35	    | 36	| 35	| 35  |
| 6	| 14	  | 45	    | 46	| 45	| 45  |
| 7	| 1	  | 56	    | 58	| 56	| 56  |
| 8	| 44	  | 69	    | 70	| 69	| 69  |
| 9	| 3	  | 82	    | 84	| 82	| 82  |
| 10	| 6	  | 95	    | 98	| 95	| 95  |
| 11	| 2	  | 109	    | 112	| 109	| 109 |
| 12	| 52	  | 125	    | 126	| 125	| 125 |
| 13	| 2	  | 140	    | 142	| 140	| 140 |
| 14	| 6	  | 156	    | 158	| 156	| 156 |
| 15	| 2	  | 172	    | 174	| 172	| 172 |
| 16	| 4	  | 190	    | 192	| 190	| 190 |
| 17	| 9	  | 208	    | 210	| 208	| 208 |
| 18	| 4	  | 226	    | 228	| <span style="background:lightgreen">226</span>	| 227 |
| 19	| 8	  | 243	    | 248	| 243	| 243 |
| 20	| 7	  | 264	    | 268	| 264	| 264 |
| 21	| 2	  | 282	    | 288	| <span style="background:lightgreen">282</span>	| 283 |
| 22	| 2	  | 300	    | 308	| <span style="background:lightgreen">300</span>	| 302 |
| 23	| 1	  | 322	    | 328	| 323	| <span style="background:lightgreen">322</span> |
| 24	| 6	  | <span style="background:pink">340</span>	    | 348	| 348	| <span style="background:lightgreen">342</span> |
| 25	| 2	  | <span style="background:pink">363</span>	    | 370	| 367	| <span style="background:lightgreen">365</span> |
| 26	| 5	  | 388	    | 392	| 388	| 388 |
| 27	| 2	  | 409	    | 414	| 410	| <span style="background:lightgreen">409</span> |
| 28	| 16	  | <span style="background:pink">435</span>	    | 436	| <span style="background:lightgreen">436</span>	| 437 |
| 29	| 1	  | 454	    | 460	| 459	| <span style="background:lightgreen">454</span> |
| 30	| 2	  | 480	    | 484	| 483	| <span style="background:lightgreen">480</span> |
| 31	| 1	  | 504	    | 508	| 507	| <span style="background:lightgreen">504</span> |
| 32	| 12	  | 528	    | 532	| 532	| <span style="background:lightgreen">528</span> |
| 33	| 1	  | <span style="background:pink">553</span>	    | 558	| 557	| <span style="background:lightgreen">555</span> |
| 34	| 17	  | 581	    | 584	| 581	| 581 |
| 35	| 4	  | 603	    | 610	| 609	| <span style="background:lightgreen">603</span> |
| 36	| 2	  | <span style="background:pink">629</span>	    | 636	| 635	| <span style="background:lightgreen">634</span> |
| 37	| 4	  | <span style="background:pink">659</span>	    | 662	| 661	| 661 |
| 38	| 16	  | <span style="background:pink">684</span>	    | 688	| 686	| 686 |
| 39	| 598	  | 713	    | 714	| 713	| 713 |
| 40	| 5	  | 740	    | 740	| <span style="background:lightgreen">740</span>	| 743 |
| 41	| 1	  | 765	    | 768	| 767	| <span style="background:lightgreen">765</span> |
| 42	| 5580	  | 795	    | 796	| 795	| 795 |
| 43	| 3528	  | 822	    | 824	| 822	| 822 |
| 44	| 1	  | 843	    | 852	| 851	| <span style="background:lightgreen">843</span> |
| 45	| 1	  | 880	    | 880	| <span style="background:lightgreen">880</span>	| 887 |
| 46	| 68	  | 909	    | 910	| 909	| 909 |
| 47	| unknown | unknown | 940	| 939	| 939 |
| 48	| unknown | unknown | 970	| 968	| 968 |
| 49	| unknown | unknown | 1000	| 999	| 999 |
| 50	| unknown | unknown | 1030	| <span style="background:lightgreen">1030</span>	| 1037	|
| 51	| unknown | unknown | 1062	| 1061	| 1061	|
| 52	| unknown | unknown | 1094	| 1093	| 1093	|
| 53	| unknown | unknown | 1126	| 1124	| <span style="background:lightgreen">1122</span>	|
| 54	| unknown | unknown | 1158	| 1157	| 1157	|
| 55	| unknown | unknown | 1190	| 1189	| <span style="background:lightgreen">1186</span>	|
| 56	| unknown | unknown | 1222	| 1221	| <span style="background:lightgreen">1217</span>	|
| 57	| unknown | unknown | 1254	| 1253	| <span style="background:lightgreen">1250</span>	|
| 58	| unknown | unknown | 1286	| 1285	| <span style="background:lightgreen">1281</span>	|
| 59	| unknown | unknown | 1318	| 1317	| <span style="background:lightgreen">1311</span>	|
| 60	| unknown | unknown | 1350	| <span style="background:lightgreen">1350</span>	| 1359	|
| 61	| unknown | unknown | 1384	| 1383	| 1383	|
| 62	| unknown | unknown | 1418	| 1417	| 1417	|
| 63	| unknown | unknown | 1452	| 1451	| 1451	|

