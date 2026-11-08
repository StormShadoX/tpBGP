# Commandes du TP BGP (GNS3, c7200)

Interfaces utilisées : `FastEthernet0/0`, `Serial1/0`, `Serial1/1`. Vérifier le nom du prompt (`CR#`, `ISP1#`, `ISP2#`) avant de coller chaque bloc.

## Étape 1 : Configuration de base

### CR

```
enable
configure terminal
hostname CR
no ip domain-lookup
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit
interface Serial1/0
 ip address 10.10.10.1 255.255.255.0
 no shutdown
end
```

### ISP1

```
enable
configure terminal
hostname ISP1
no ip domain-lookup
interface Serial1/0
 ip address 10.10.10.2 255.255.255.0
 no shutdown
exit
interface Serial1/1
 ip address 172.16.1.1 255.255.255.0
 no shutdown
exit
interface loopback0
 ip address 192.168.100.1 255.255.255.0
end
```

### ISP2

```
enable
configure terminal
hostname ISP2
no ip domain-lookup
interface Serial1/0
 ip address 172.16.1.2 255.255.255.0
 no shutdown
exit
interface loopback0
 ip address 192.168.200.1 255.255.255.0
end
```

### PC1 (VPCS)

```
ip 192.168.1.2/24 192.168.1.1
save
```

### Tests de connectivité (avant les routes)

```
CR# ping 10.10.10.2
CR# ping 172.16.1.2
PC1> ping 10.10.10.2
```

- CR atteint ISP2 ? **Non** (aucune route vers 172.16.1.0/24).
- L'hôte client atteint ISP1 ? **Non** (pas encore de route par défaut sur CR).

### Vérification des interfaces

```
show ip interface brief
```

## Étape 2 : Route par défaut et route statique

### CR

```
configure terminal
ip route 0.0.0.0 0.0.0.0 10.10.10.2
end
```

### ISP1

```
configure terminal
ip route 192.168.1.0 255.255.255.0 10.10.10.1
end
```

### Test

```
PC1> ping 10.10.10.2
```

## Étape 3 : Configuration BGP

CR n'exécute **pas** BGP.

### ISP1 (AS 100)

```
configure terminal
router bgp 100
 neighbor 172.16.1.2 remote-as 200
 network 192.168.1.0
 network 192.168.100.0
end
write memory
```

### ISP2 (AS 200)

```
configure terminal
router bgp 200
 neighbor 172.16.1.1 remote-as 100
 network 192.168.200.0
end
write memory
```

### Vérification de la session (attendre 30 à 60 secondes)

```
ISP1# show ip bgp summary
ISP2# show ip bgp summary
```

`State/PfxRcd` doit afficher un nombre (1 sur ISP1, 2 sur ISP2), pas `Idle` ni `Active`.

## Étape 4 : Tables de routage

```
ISP2# show ip route
ISP1# show ip route
CR# show ip route
```

| Question | Réponse |
|---|---|
| ISP2 : 192.168.1.0 dans la table ? | Oui |
| Lettre à gauche de 192.168.1.0 | `B` |
| Signification | Route apprise par BGP |
| ISP2 : 192.168.100.0 dans la table ? | Oui |
| Routeur qui a annoncé 192.168.1.0 | ISP1 (AS 100) |
| ISP1 : réseau annoncé par ISP2 | 192.168.200.0/24 |
| ISP1 : comment 192.168.1.0 a-t-il été appris ? | Route statique (`S`) vers 10.10.10.1, injectée dans BGP par `network` |
| ISP1 annonce-t-il des réseaux au client ? | Non |
| CR : pourquoi pas 192.168.100.0 ni 192.168.200.0 ? | CR n'exécute pas BGP, il a seulement une route par défaut (`S*`) |

## Étape 5 : Vérification de la connectivité

```
PC1> ping 192.168.200.1
ISP2# ping 192.168.1.2
ISP2# ping 192.168.1.2 source loopback0
```

Pour un test plus fiable sur l'émulateur :

```
PC1> ping 192.168.200.1 -c 20
ISP2# ping 192.168.1.2 repeat 50
```

## Étape 6 : Tables BGP

```
ISP1# show ip bgp
ISP2# show ip bgp
```

Sur ISP1 :

- `192.168.1.0` : Next Hop `10.10.10.1`, Weight `32768`
- `192.168.100.0` : Next Hop `0.0.0.0`, Weight `32768`
- `192.168.200.0` : Next Hop `172.16.1.2`, chemin `200`

## Étape 7 : Remarque générale

ISP1 n'annonce aucun réseau au routeur client, car CR n'est pas un voisin BGP d'ISP1 : aucune commande `neighbor` n'existe entre eux, et CR utilise seulement une route par défaut statique.

## Dépannage

| Symptôme | Vérification |
|---|---|
| Interface série `down/down` | `no shutdown` des deux côtés, bon câblage GNS3 |
| Session BGP `Idle` ou `Active` | `neighbor`, `remote-as`, ping entre 172.16.1.1 et 172.16.1.2 |
| ISP2 ne peut pas répondre au LAN client | `show ip route static` sur ISP1 : route vers 192.168.1.0/24 présente ? |
| Pertes de paquets irrégulières | Appliquer l'Idle-PC finder, surveiller le CPU (`dynamips`) |
| BGP configuré par erreur sur CR | `no router bgp 100` puis `write memory` |

## Nettoyage d'un routeur

```
erase startup-config
reload
```

## Sauvegarde

```
write memory
```
