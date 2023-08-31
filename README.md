# Network-Project
Projet Network CCNA - Dynamic Multipoint Virtual Private Network

*/ Projet réalisé en collaboration avec un camarade de classe dans le cadre d'un cours optionnel sur la certification CCNA */


Rapport Projet DMVPN
Anthony Gachet – Vincenzo Ceccon


------ Etape 1: Documentation

Pour commencer notre travail de documentation, nous nous sommes rendus sur la page présente sur l’énoncé du projet :
https://www.cisco.com/c/en/us/products/collateral/security/dynamic-multipoint-vpn-dmvpn/data_sheet_c78-468520.html
Cette page nous a permis de comprendre en grandes lignes ce qu’était le DMVPN et son utilisation. Nous avons appris les technologies qui étaient comprises dans cet outil comme le Multipoint GRE, le NHRP (Next Hop Resolution Protocol) et l’IPSec qui lui n’est pas obligatoire mais recommandé.

Afin d’approfondir nos connaissances et surtout clarifier les notions de phases. Nous avons étudié l’article suivant :
https://network-insight.net/2015/02/03/design-guide-dmvpn-phases
Nous avons appris que pendant la phase 1, tout le trafic passe par le hub, il n’y a pas de connexion directe  de spoke à spoke. 
La phase 2 permet l’établissement de tunnels entre les spokes et ainsi la communication sans passer par le hub.

La vidéo suivante nous a permis de comprendre comment bien configurer les routeurs :
https://www.youtube.com/watch?v=68Raa0FWNkg

Lors de la documentation et après l’établissement du plan réseau prototype, nous en avons profité pour écrire dans un document Word les configurations du mGRE et NHRP pour les spokes et le hub.

Pour perfectionner nos connaissances, nous avons consulté ce document :

https://learningnetwork.cisco.com/s/article/dmvpn-concepts-amp-configuration

------ Etape 2: Etablissement du plan réseau prototype

Dans notre projet, nous avons choisi d’utiliser trois spokes, un hub et un routeur pour jouer le rôle de l’iISP. Le choix d’utiliser trois spokes était simplement pour en faire un peu plus que ce qui était demandé, mais le projet pourrait très bien fonctionner avec deux spokes, dans le sens que tous les tests pour voir si le DMVPN fonctionne pourraient être effectués.

------ Etape 3: Mise en place

Phase 1

Que comprend la phase 1 ?
-	La phase 1 consiste à des tunnels mGRE sur le hub, qui peut dynamiquement enregistrer des nouveaux spokes. Tout le traffic, même le traffic spoke to spoke, passe par le hub. La configuration du tunnel GRE sur les spoke est une configuration point to point (spoke à hub). 

Pour la mise en pratique, nous avons effectué le câblage selon le plan réseau. Nous avons ensuite configuré les interfaces et réalisé les tests de pings pour s’assurer que la connectivité était en état. Une fois ces tests concluants, nous avons implémenté le routage dynamique. Dans notre projet nous avons décidé d’utiliser l’OSPF. Une fois ce protocole bien mis en place et fonctionnel, nous nous sommes lancés sur la configuration du DMVPN, c’est-à-dire du mGRE en combinaison avec le NHRP. Pour cela nous avons utilisé les configurations écrites pendant la documentation. Lorsque nous voulions obtenir les informations sur le statut du DMVPN avec la commande show dmvpn, rien ne s’affichait et le trafic entre le hub et un pc situé derrière un spoke ne passait pas par le tunnel. Nous avons rajouté deux commandes dans le spoke qui ont réglé le premier problème, mais pas le deuxième.
Pour le deuxième problème, la configuration venait de l’OSPF. Nous avions rajouté l’OSPF sur l’ISP et rajouté les réseaux allant des spokes à l’ISP et le réseau du hub à l’ISP. Cela ne fonctionnait pas, car le trafic passait directement par la meilleure route, celle configurée (de par les réseaux intermédiaires qui y était rajouté) dans l’OSPF.

Phase 2

Que comprend la phase 2 ?

-	La phase 2 du DMVPN permet aux spokes de pouvoir communiquer entre eux directement, sans que tout le trafic passe par le hub. Pour cela, quand un spoke veut communiquer avec un autre spoke, il envoie un paquet NHRP Resolution Request au Hub/NHS. Le Hub lui répond avec un paquet NHRP Resolution Reply depuis son cache et le Spoke peut alors connaitre l’adresse NBMA d’un autre Spoke et le contacter directement. Un tunnel est alors dynamiquement créé entre un spoke et un autre.
De gros problèmes au niveau de l’OSPF sont survenus au moment de rebrancher le projet en début de cours, cela nous a couté environ deux heures. Changer l’OSPF en broadcast comme recommandé n’a pas réglé le problème. Nous avons recherché pendant un long moment, fait de nombreux changements de configurations (autant dans l’OSPF que dans l’interface du tunnel) et l’OSPF n’était toujours pas stable. La solution était que la configuration du NHRP sur deux spokes n’étaient pas bien configurée, ils n’avaient pas l’option “ip nhrp map multicast IP_HUB”. De plus, changer l’OSPF en rajoutant l’option “ip ospf network point-to-multpoint" a assuré la stabilité de l’OSPF.
Pour l’établissement de la phase 2, nous avons changé les tunnels sur les spokes en tunnels multipoint (comme cela était configuré sur le hub). Cela permet donc aux spokes de plus être en mode point to point (uniquement vers le hub), mais de pouvoir créer dynamiquement des tunnels avec les autres spokes (après avoir reçu les informations NHRP depuis le hub).

Phase 3

Que comprend la phase 3 ?

-	Lors de la phase 3, le hub redirige les demandes NHRP avec un NHRP redirect. Ce message est envoyé sur l’interface où il a reçu la demande, donc le spoke à l’origine de la demande. Le spoke envoie ensuite un NHRP Resolution Request qui est envoyé à tous. Le spoke qui est le destinataire renvoie un NHRP Reply avec ses informations. Enfin, grâce à la commande NHRP Shortcut, le spoke d’origine met à jour sa table NHRP.

Pour établir cette phase, les commandes sont peu nombreuses. Il suffit d’ajouter un « ip nhrp redirect » sur le hub, pour dire « ce n’est plus moi qui s’occupe de fournir les informations NHRP ». Sur tous les spokes, il faut insérer la commande « ip nhrp shortcut », afin qu’une fois que le spoke à reçu les informations NHRP du spoke dont il a demandé les infos, il puisse les enregistrer dans sa propre table NHRP.

Le dernier problème que nous avions avec la phase 3 est que dans la table de routage des spokes, nous avions toujours le via « IP_TUNNEL_HUB » malgré la phase 3 bien configurée. Normalement nous devrions avoir via « IP_TUNNEL_SPOKE » avec l’adresse du spoke, car le routeur sait désormais que la meilleure route passe par le spoke destination et le next hop est désormais le spoke destination. Hors pour nous le via était toujours le hub. Après de nombreuses recherches, nous avons trouvé que, parce que nous étions en OSPF point-to-multipoint, cela était normal. Si nous étions en OSPF broadcast, nous aurions en effet du voir le via spoke destination. Mais avec le OSPF point-to-multipoint, c’est le symbole « % » à côté du type de route qui nous indique qu’il y a un « NHO », un Next Hop Override. Si on execute un « sh ip route next-hop-override” on peut désormais voir que le via est override et que c’est bien l’adresse du spoke qui est comme NHO. Cela nous prouve que la phase 3 fonctionne

------ Etape 4 : Tests

Nous n'avons pas inclus les tests basiques de connectivité.

Tests pour la construction de tunnels dynamiques entre deux spoke :
1.	Le premier traceroute entre le spoke 1 et le spoke 3 passe par le hub, parce que le tunnel dynamique entre le spoke 1 et le spoke 3 n’est pas encore créé.

-	Spoke1#traceroute 172.16.48.4
-	Type escape sequence to abort.
-	Tracing the route to 172.16.48.4
-	VRF info: (vrf in name/id, vrf out name/id)
-	  1 172.16.48.1 0 msec 0 msec 0 msec
-	  2 172.16.48.4 4 msec 0 msec *

2.	Le suivant ne passe plus par le hub, car le tunnel est désormais créé.

-	Spoke1#traceroute 172.16.48.4
-	Type escape sequence to abort.
-	Tracing the route to 172.16.48.4
-	VRF info: (vrf in name/id, vrf out name/id)
-	  1 172.16.48.4 0 msec 0 msec *
-	Spoke1#

3.	On peut voir ce tunnel dynamique dans le spoke 3 grace aux commandes suivantes :

-	Spoke1#sh ip nhr
-	172.16.48.1/32 via 172.16.48.1
-	   Tunnel1 created 02:51:49, never expire
-	   Type: static, Flags: used
-	   NBMA address: 10.136.48.1
-	172.16.48.4/32 via 172.16.48.4
-	   Tunnel1 created 00:02:20, expire 00:07:39
-	   Type: dynamic, Flags: router used nhop rib nho
-	   NBMA address: 10.136.51.2

-	Spoke1#sh dmvpn
-	Legend: Attrb --> S - Static, D - Dynamic, I – Incomplete
-	        N - NATed, L - Local, X - No Socket
-	        T1 - Route Installed, T2 - Nexthop-override
-	        C - CTS Capable, I2 – Temporary
-	        # Ent --> Number of NHRP entries with same NBMA peer
-	        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
-	        UpDn Time --> Up or Down Time for a Tunnel
-	=====================================================================
-	Interface: Tunnel1, IPv4 NHRP Details
-	Type:Spoke, NHRP Peers:2,
-	
-	 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
-	 ----- --------------- --------------- ----- -------- -----
-	     1 10.136.48.1         172.16.48.1    UP 02:48:36     S
-	     1 10.136.51.2         172.16.48.4    UP 00:02:37   DT2
-	Spoke1#

Voici un show ip nhrp shortcut qui nous montre le fonctionnement de la phase 3 et la communication qui ne passe plus par le hub mais directement au spoke :

Spoke1#sh ip nh
Spoke1#sh ip nhrp sh
Spoke1#sh ip nhrp shortcut
172.16.48.4/32 via 172.16.48.4
   Tunnel1 created 00:07:03, expire 00:02:56
   Type: dynamic, Flags: router nhop rib nho
   NBMA address: 10.136.51.2
192.168.3.0/24 via 172.16.48.4
   Tunnel1 created 00:07:03, expire 00:02:56
   Type: dynamic, Flags: router rib nho
   NBMA address: 10.136.51.2
Spoke1#


