## Installation de sensu

L'intallation de sensu peut se faire en suivant la [Sensu docs](https://docs.sensu.io/sensu-go/latest/operations/deploy-sensu/install-sensu/)

---
## Configs

### IP statiques

##### Ubuntu server
Créer un fichier `/etc/netplan/01-ens37-static.yaml`

```txt
network:
  version: 2
  renderer: networkd
  ethernets:
    ens37:
      dhcp4: no
      addresses:
        - 192.168.131.10/24

```

- `sudo chmod 600 /etc/netplan/01-ens37-static.yaml`

- `sudo netplan apply`

##### Debian Xfce
  
  Ajouter ça dans le fichier `/etc/network/interfaces`
  `
  
```txt
	auto ens37
	iface ens37 inet static
	address 192.168.131.11
	netmask 255.255.255.0
```

- `sudo /etc/init.d/networking restart`

*N'oublie pas de changer l'url de l'api si jamais tu rollback*

---
### Téléchargement des assets et configuration des checks

 Les détails de chaque commandes peuvent être trouvés sur le site [bonsai](https://bonsai.sensu.io/) de Sensu.

 Nous devons d'abord créer une subscription nommée `system` et y intégrer le serveur web : `sensuctl entity update <entity_name>`

##### Check pour le CPU

- Ajouter les métadonnées de check CPU sur sensu backend
	`sensuctl asset add sensu-plugins/sensu-plugins-cpu-checks:4.1.0 -r cpu-checks-plugins`
- Ajouter le ruby runtime puisque check CPU fonctionne avec ruby
	`sensuctl asset add sensu/sensu-ruby-runtime:0.0.10 -r sensu-ruby-runtime`

- Créer le check: 
```txt
sensuctl check create check_cpu \
--command 'check-cpu.rb -w 75 -c 90' \
--interval 30 \
--subscriptions system \
--runtime-assets cpu-checks-plugins,sensu-ruby-runtime
```
##### Check pour la RAM

- Ajouter les métadonnées de memory and swap usage checks `sensuctl asset add sensu/check-memory-usage -r check-memory-usage`

- Créer le check
```text
sensuctl check create check_memory \
--command 'check-memory-usage -w 75 -c 90' \
--interval 30 \
--subscriptions system \
--runtime-assets check-memory-usage
```

#### Check du disk

- Ajouter les métadonnées de check disk `sensuctl asset add sensu/check-disk-usage -r check-disk-usage`
- Créer le check
```text
sensuctl check create check_disk \
--command "check-disk-usage -w 50 -c 75" \
--interval 60 \
--subscriptions system \
--runtime-assets check-disk-usage
```

#### Check du service apache2

- `sensuctl asset add sensu-plugins/sensu-plugins-process-checks -r sensu-process-checks`

```txt
sensuctl check create check-apache2 \
--command 'check-process.rb -p apache2' \
--interval 60 \
--subscriptions system \
--runtime-assets sensu-process-checks,sensu-ruby-runtime
```

---
### Configuration du serveur SMTP avec Gmail et de tout ce qui a rapport avec un mail

#### Ajouter un asset pour le handler mail

```text
sensuctl asset add sensu/sensu-email-handler -r email-handler
```

Suite [ici](https://github.com/MansoorMajeed/devops-from-scratch/blob/master/episodes/34-monitoring-5-getting-email-alerts.md#vm-monitoring-5---receiving-email-alerts)

#### Keep alive email

 Pour recevoir un mail lorsque le serveur apache est down

- Editer la config de l'entité en utilisant `sensuctl edit entity debian`.
- Ajouter :
```txt
keepalive_handlers: 
- email
```

#### Template du mail

*Template généré par l'IA*

- Créer un fichier pour le template --> `sudo nano /etc/sensu/email_template.html` et ajouter ceci :

```txt
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
</head>
<body style="font-family: Arial, sans-serif; color: #333;">

    <div style="background-color: #eee; font-size: 10px; padding: 5px; border-bottom: 1px solid #ccc;">
        DEBUG INFO: Status={{ .Check.Status }} | State={{ .Check.State }} | Name={{ .Check.Name }}
    </div>

    <h2>
        Alerte sur {{ .Entity.Name }}
    </h2>

    <div style="padding: 15px; border: 1px solid #ddd; border-radius: 5px;">
        
        <p><strong>Statut :</strong> 
            {{ if eq .Check.Status 2 }}
                <span style="color: red; font-size: 1.2em; font-weight: bold;">CRITICAL (Panne)</span>
            {{ else if eq .Check.Status 1 }}
                <span style="color: orange; font-size: 1.2em; font-weight: bold;">WARNING (Attention)</span>
            {{ else }}
                <span style="color: green; font-size: 1.2em; font-weight: bold;">RESOLVED (Rétabli)</span>
            {{ end }}
        </p>

        <p><strong>Serveur :</strong> {{ .Entity.Name }}</p>
        <p><strong>Check :</strong> {{ .Check.Name }}</p>
    </div>

    <h3>Détails techniques :</h3>
    <div style="background-color: #2d2d2d; color: #fff; padding: 10px; font-family: monospace; border-radius: 4px;">
        {{ .Check.Output }}
    </div>

</body>
</html>
```

- Update le handler mail en faisant:
	- `sensuctl handler update email`

Arrivée sur la ligne `Command`, faire:

```txt
sensu-email-handler -f <mail_de_l'expéditeur> -t <mail_du_destinataire> -s smtp.gmail.com -u <mail_de_l'expéditeur> -p 'Mot_De_Passe' -T /etc/sensu/email_template.html -S "[SENSU] Alerte sur {{ .Entity.Name }}"
```

---
### Configuration du DNS

Les deux configurations doivent se faire en mode administrateur/root. Ajouter ces lignes dans 
`C:\Windows\System32\drivers\etc\hosts` pour **Windows** et `/etc/hosts` sous **Linux**

```text
192.168.56.10   sensu.lab
192.168.56.11   web.lab
```


---

## C'est quoi Sensu ?

C'est un outil de monitoring basé sur Go qui supporte le monitoring par push (Ce sont les agents qui envoient les données au backend). Par défaut, il utilise le push c'est à dire que ce sont les agents qui envoient les métriques mesurées au backend sans qu'il ait besoin de demander lui même.

### Architecture de Sensu

![](attachments/Pasted%20image%2020251215205319.png)

*Explication de l'infra à faire*

- **sensuctl** --> CLI qui utilise les API HTTP de Sensu-Backend pour ajouter supprimer et modifier des paramètres.

- **browser** --> Le navigateur qui communique avec l'UI web de Sensu-Backend
  
- **statsD listener** -->**collecte les métriques des applis et les stocke temporairement dans l’agent**, puis l’**Agent API** les envoie au backend. Il **ne push pas directement** au backend
  
- **Agent API** --> C'est le cœur de l'agent. Il exécute les checks demandés par le backend et push les métriques collectées par le **statsD listener**
  
- **HTTP API** --> Permet de faire un CRUD sur les objets du backend Sensu
  
- **WebSocket API** --> Reçois en temps réel les métriques collectées par le Sensu-Agent
  
- **Elbedded etcd strore** --> Une base clé-valeur distribuée.

### Objets et Entités 

Dans Sensu, les Entities représentent les instances surveillées (machines, containers, services), tandis que les Objects sont les définitions ou configurations (checks, handlers, silences) qui s’appliquent à ces entities.

### Facts

- Le token JWT peut finir par expirer. Si c'est le cas, et que l'on veut utiliser `sensuctl`, il faut se réauth en faisant : `sensuctl configure`

- **Agent** --> C'est un programme qui 

- **Check** --> C'est une vérification faites par un agent

- **Handler** --> Le programme qui prend en entrée le résultat du check et réagit en conséquence.

- **Asset** --> C'est une archive compressée qui contient du code qui sera utiliser par les check, handler, template etc... Lien vers les [assets](https://bonsai.sensu.io/assets). L'asset lorsqu'il est ajouté n'est pas automatiquement installé sur une machine. *Finir les explications*

-  **Cluster Sensu** --> un ensemble de trois nodes backend qui bossent ensemble
  Plus d'infos sur les clusters dans la [doc](https://docs.sensu.io/sensu-go/latest/operations/deploy-sensu/cluster-sensu/).

### CheatSheet

 La [doc](https://docs.sensu.io/sensu-go/latest/sensuctl/) pour sensuctl

Les configs des agents se trouvent dans `/etc/sensu/agent.yml`

- Lister les entity --> `sensuctl entity list`
- Update une entity --> `sensuctl entity update <ENTITY>`

#### Supprimer un agent

- sudo systemctl stop sensu-agent
- sudo systemctl disable sensu-agent
- sudo apt remove sensu-go-agent
- sudo yum remove sensu-go-agent
- sensuctl entity list
- sensuctl entity delete <NOM_DE_L_AGENT>

#### cmd 

- `htop` --> Pour monitorer la RAM d'une machine
- `df` --> Pour monitorer le disque
- `stress -c 2` --> Pour solliciter les deux coeurs du CPU du serveur et déclencher une alerte dans notre infra
- `stress --vm 1 --vm-bytes 1G --timeout 60s` --> Pour saturer la RAM et déclencher un warning.
- `dd if=/dev/zero of=fill_disk.bin bs=1M count=10240 status=progress` --> Ajouter 10 Go de stockage au disque
	- supprimer avec `rm` et utilisé `sync`.
  
  
