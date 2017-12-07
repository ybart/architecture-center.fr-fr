---
title: "Exécuter une machine virtuelle Linux dans Azure"
description: "Comment exécuter une machine virtuelle Linux sur Azure, en privilégiant l’évolutivité, la résilience, la gestion et la sécurité."
author: telmosampaio
ms.date: 11/16/2017
pnp.series.title: Linux VM workloads
pnp.series.next: multi-vm
pnp.series.prev: ./index
ms.openlocfilehash: f538958be934ad2e9ea8d53791814b1e963c1a20
ms.sourcegitcommit: 115db7ee008a0b1f2b0be50a26471050742ddb04
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 11/17/2017
---
# <a name="run-a-linux-vm-on-azure"></a>Exécuter une machine virtuelle Linux dans Azure

Cette architecture de référence présente un ensemble de pratiques éprouvées pour l’exécution d’une machine virtuelle Linux dans Azure. Elle contient des recommandations pour le provisionnement de la machine virtuelle et des composants réseau et de stockage. Cette architecture peut être utilisée pour exécuter une instance unique de machine virtuelle et sert de base aux architectures plus complexes telles que les applications multiniveaux. [**Déployez cette solution.**](#deploy-the-solution)

![[0]][0]

*Téléchargez un [fichier Visio][visio-download] contenant ce diagramme d’architecture.*

## <a name="architecture"></a>Architecture

L’approvisionnement d’une machine virtuelle Azure requiert des composants supplémentaires, tels que les ressources de calcul, de mise en réseau et de stockage.

* **Groupe de ressources.** Un [*groupe de ressources*][resource-manager-overview] est un conteneur qui héberge les ressources associées. En général, vous devez regrouper les ressources d’une solution en fonction de leur durée de vie et de la personne qui doit gérer les ressources. Pour une charge de travail de machine virtuelle unique, vous souhaiterez peut-être créer un seul groupe de ressources pour toutes les ressources.
* **Machine virtuelle**. Vous pouvez approvisionner une machine virtuelle issue d’une liste d’images publiées, d’une image managée personnalisée ou d’un fichier de disque dur virtuel (VHD) chargé(e) dans Stockage Blob Azure. Azure prend en charge l’exécution de plusieurs distributions Linux populaires, y compris CentOS, Debian, Red Hat Enterprise, Ubuntu et FreeBSD. Pour en savoir plus, voir [Azure et Linux][azure-linux].
* **Disque du système d’exploitation.** Le disque du système d’exploitation est un disque dur virtuel stocké dans [Stockage Azure][azure-storage], donc il persiste même lorsque l’ordinateur hôte est arrêté. Pour les machines virtuelles Linux, le disque du système d’exploitation est `/dev/sda1`.
* **Disque temporaire** La machine virtuelle est créée avec un disque temporaire. Ce disque est stocké sur un lecteur physique de l’ordinateur hôte. Il n’est *pas* enregistré dans Stockage Azure et peut être supprimé lors des redémarrages ou d’autres événements de cycle de vie de la machine virtuelle. N’utilisez ce disque que pour des données temporaires, telles que des fichiers de pagination ou d’échange. Pour les machines virtuelles Linux, le disque temporaire est `/dev/sdb1`. Il est monté sur `/mnt/resource` ou `/mnt`.
* **Disques de données.** Un [disque de données][data-disk] est un disque dur virtuel persistant utilisé pour les données d’application. Les disques de données sont stockés dans Stockage Azure, comme le disque du système d’exploitation.
* **Réseau virtuel (VNet) et sous-réseau.** Chaque machine virtuelle Azure est déployée dans un réseau virtuel qui peut être segmenté en plusieurs sous-réseaux.
* **Adresse IP publique.** Une adresse IP publique est nécessaire pour communiquer avec la machine virtuelle, par exemple par le biais du protocole SSH.
* **Interfaces réseau (NIC)**. La carte d’interface réseau assignée permet à la machine virtuelle de communiquer avec le réseau virtuel.
* **Groupe de sécurité réseau**. Les [groupes de sécurité réseau][nsg] sont utilisés pour autoriser ou refuser le trafic réseau vers une ressource réseau. Vous pouvez associer un NSG à une carte réseau individuelle ou à un sous-réseau. Si vous l’associez à un sous-réseau, les règles du NSG s’appliquent à toutes les machines virtuelles de ce sous-réseau.
* **Diagnostics.** La journalisation des diagnostics est essentielle à la gestion et au dépannage de la machine virtuelle.

## <a name="recommendations"></a>Recommandations

Cette architecture présente les recommandations de base pour l’exécution d’une machine virtuelle Linux dans Azure. Nous vous déconseillons toutefois d’utiliser une seule et même machine virtuelle pour les charges de travail critiques, car ce faisant, un point de défaillance unique est créé. Pour bénéficier d’une disponibilité plus élevée, vous devez déployer plusieurs machines virtuelles dans un [groupe à haute disponibilité][availability-set]. Pour plus d’informations, voir [Running multiple VMs on Azure (Exécution de plusieurs machines virtuelles sur Azure)][multi-vm]. 

### <a name="vm-recommendations"></a>Recommandations pour les machines virtuelles

Azure propose de nombreuses tailles de machines virtuelles. Le [Stockage Premium][premium-storage] est recommandé en raison de ses hautes performances et d’une faible latence, et il est [pris en charge par des tailles de machine virtuelle spécifiques][premium-storage-supported]. Choisissez l’une de ces tailles, sauf si vous disposez d’une charge de travail spécialisée telle qu’un système de calcul hautes performances. Pour plus d’informations, consultez [Tailles de machines virtuelles][virtual-machine-sizes].

Si vous déplacez une charge de travail vers Azure, commencez par choisir la taille de machine virtuelle qui correspond le mieux à vos serveurs locaux. Mesurez ensuite les performances de votre charge de travail réelle en termes de processeur, de mémoire et d’opérations d’entrée/sortie par seconde du disque, puis ajustez la taille selon vos besoins. Si vous avez besoin de plusieurs cartes réseau (NIC) pour votre machine virtuelle, notez que le nombre maximal de cartes NIC est défini pour chaque [taille de machine virtuelle][vm-size-tables].

Lorsque vous approvisionnez des ressources Azure, vous devez spécifier une région. En général, choisissez une région la plus proche possible de vos utilisateurs internes ou de vos clients. Sachez toutefois que certaines tailles de machine virtuelle ne sont pas disponibles dans toutes les régions. Pour en savoir plus, consultez [Services par région][services-by-region]. Pour obtenir la liste des tailles de machine virtuelle disponibles dans une région spécifique, exécutez la commande suivante dans l’interface de ligne de commande (CLI) Azure :

```
az vm list-sizes --location <location>
```

Pour en savoir plus sur le choix d’une image de machine virtuelle publiée, voir [Comment rechercher des images de machine virtuelle Linux][select-vm-image].

Permet la surveillance et le diagnostic, avec notamment des indicateurs d’intégrité de base, des journaux d’infrastructure de diagnostic et des [diagnostics de démarrage][boot-diagnostics]. Les diagnostics de démarrage peuvent vous aider à identifier le problème de démarrage si votre machine virtuelle refuse de démarrer. Pour en savoir plus, voir [Activation de la surveillance et des diagnostics][enable-monitoring].  

### <a name="disk-and-storage-recommendations"></a>Recommandations pour le disque et le stockage

Pour optimiser les performances d’E/S du disque, nous vous recommandons le [Stockage Premium][premium-storage], qui stocke les données sur des disques SSD. Le coût est basé sur la capacité du disque approvisionné. Le nombre d’E/S par seconde et le débit (c’est-à-dire le taux de transfert des données) dépendent également de la taille du disque. Lorsque vous approvisionnez un disque, vous devez donc tenir compte des trois facteurs : capacité, E/S par seconde et débit. 

Nous vous recommandons également d’utiliser des [disques managés](/azure/storage/storage-managed-disks-overview). Les disques managés ne nécessitent pas de compte de stockage. Il vous suffit de spécifier leur taille et leur type, puis de les déployer en tant que ressource hautement disponible.

Si vous utilisez des disques non gérés, créez des comptes de stockage Azure distincts pour chaque machine virtuelle pour stocker les disques durs virtuels (VHD), afin d’éviter d’atteindre les [limites d’E/S par seconde][vm-disk-limits] des comptes de stockage.

Ajoutez un ou plusieurs disques de données. Lorsque vous créez un disque dur virtuel, il n’est pas formaté. Connectez-vous à la machine virtuelle pour formater le disque. Si vous n’utilisez pas de disques managés et que vous avez un grand nombre de disques de données, n’oubliez pas les limites d’E/S du compte de stockage. Pour en savoir plus, voir les [limites du nombre de disques de machine virtuelle][vm-disk-limits].

Dans l’interpréteur de commandes Linux, les disques de données sont affichés en tant que `/dev/sdc`, `/dev/sdd`, et ainsi de suite. Vous pouvez exécuter `lsblk` pour répertorier les périphériques de bloc, y compris les disques. Pour utiliser un disque de données, créez une partition et un système de fichiers, puis montez le disque. Par exemple :

```bat
# Create a partition.
sudo fdisk /dev/sdc     # Enter 'n' to partition, 'w' to write the change.

# Create a file system.
sudo mkfs -t ext3 /dev/sdc1

# Mount the drive.
sudo mkdir /data1
sudo mount /dev/sdc1 /data1
```

Lorsque vous ajoutez un disque de données, un numéro d’unité logique (LUN) lui est attribué. Vous pouvez également spécifier l’ID LUN, par exemple si vous remplacez un disque et souhaitez conserver le même ID LUN, ou si votre application nécessite un ID LUN spécifique. Toutefois, n’oubliez pas que les ID LUN doivent être uniques pour chaque disque.

Vous pouvez modifier le planificateur d’E/S afin d’optimiser les performances des disques SSD, les disques des machines virtuelles associées à des comptes de Stockage Premium étant de type SSD. Il est généralement recommandé d’utiliser le planificateur NOOP pour les disques SSD, mais vous devez utiliser un outil tel que [iostat] pour surveiller les performances d’E/S du disque pour votre charge de travail.

Pour maximiser les performances, créez un compte de stockage distinct destiné à héberger les journaux de diagnostic. Un compte de stockage localement redondant (LRS) standard suffit pour les journaux de diagnostic.

### <a name="network-recommendations"></a>Recommandations pour le réseau

Cette adresse IP publique peut être dynamique ou statique. Par défaut, elle est dynamique.

* Utilisez une [adresse IP statique][static-ip] si vous avez besoin d’une adresse IP non modifiable, par exemple pour créer un enregistrement A dans le DNS ou ajouter l’adresse IP dans une liste sécurisée.
* Vous pouvez également créer un nom de domaine complet (FQDN) pour l’adresse IP. Vous pouvez inscrire un [enregistrement CNAME][cname-record] dans le DNS qui pointe vers le nom de domaine complet (FQDN). Pour en savoir plus, voir [Créer un nom de domaine complet dans le Portail Azure][fqdn].

Tous les groupes de sécurité réseau contiennent un ensemble de [règles par défaut][nsg-default-rules], notamment une règle qui bloque tout le trafic Internet entrant. Les règles par défaut ne peuvent pas être supprimées, mais d’autres règles peuvent les remplacer. Pour autoriser le trafic Internet, créez des règles qui autorisent le trafic entrant vers des ports spécifiques, par exemple, le port 80 pour HTTP.

Pour activer le protocole SSH, ajoutez une règle de groupe de sécurité réseau qui autorise le trafic entrant sur le port TCP 22.

## <a name="scalability-considerations"></a>Considérations relatives à l’extensibilité

Vous pouvez réduire ou augmenter la puissance d’une machine virtuelle en en [modifiant la taille][vm-resize]. Pour une mise à l’échelle horizontale, placez plusieurs machines virtuelles derrière un équilibreur de charge. Pour plus d’informations, voir [Exécution de plusieurs machines virtuelles sur Azure pour l’extensibilité et la disponibilité][multi-vm].

## <a name="availability-considerations"></a>Considérations relatives à la disponibilité

Pour bénéficier d’une disponibilité plus élevée, déployez plusieurs machines virtuelles dans un groupe à haute disponibilité. Cela fournit également un [contrat de niveau de service][vm-sla] (SLA) supérieur.

Votre machine virtuelle peut être affectée par la [maintenance planifiée][planned-maintenance] ou la [maintenance non planifiée][manage-vm-availability]. Vous pouvez utiliser les [journaux de redémarrage de machine virtuelle][reboot-logs] pour déterminer si un redémarrage de la machine virtuelle a été provoqué par une maintenance planifiée.

Les disques durs virtuels sont stockés dans [Stockage Azure][azure-storage]. Stockage Azure est répliqué à des fins de durabilité et de disponibilité.

Pour vous protéger contre la perte accidentelle de données pendant les opérations normales (par exemple, en cas d’erreur d’un utilisateur), vous devez également implémenter des sauvegardes ponctuelles, à l’aide de [captures instantanées d’objets blob][blob-snapshot] ou d’un autre outil.

## <a name="manageability-considerations"></a>Considérations relatives à la facilité de gestion

**Groupes de ressources.** Placez les ressources étroitement associées qui partagent le même cycle de vie dans un même [groupe de ressources][resource-manager-overview]. Les groupes de ressources vous permettent de déployer et de surveiller les ressources en tant que groupe et de suivre les coûts de facturation par groupe de ressources. Vous pouvez également supprimer des ressources dans un ensemble, ce qui est très utile pour les déploiements de test. Affectez des noms de ressource explicites pour simplifier la recherche d’une ressource spécifique et comprendre son rôle. Pour plus d’informations, consultez [Conventions d’affectation de noms recommandées pour les ressources Azure][naming-conventions].

**SSH**. Avant de créer une machine virtuelle Linux, générez une paire de clés publique-privée RSA 2048 bits. Utilisez le fichier de clé publique lorsque vous créez la machine virtuelle. Pour en savoir plus, voir [Utilisation de SSH avec Linux et Mac sur Azure][ssh-linux].

**Arrêt d’une machine virtuelle.** Azure établit une distinction entre les états « arrêté » et « désalloué ». Vous payez lorsque l’état de la machine virtuelle est « arrêté », mais pas lorsque la machine virtuelle est désallouée.

Le bouton **Arrêter** du portail Azure désalloue la machine virtuelle. Si vous arrêtez la machine virtuelle par le biais du système d’exploitation pendant que vous êtes connecté, la machine virtuelle est arrêtée mais *non* désallouée. Vous serez donc toujours facturé.

**Suppression d’une machine virtuelle.** La suppression d’une machine virtuelle n’entraîne pas celle des disques durs virtuels. Vous pouvez donc supprimer la machine virtuelle, sans risque de perdre des données. Toutefois, vous serez toujours facturé pour le stockage. Pour supprimer le disque dur virtuel, supprimez le fichier de [Stockage Blob][blob-storage].

Pour éviter toute suppression accidentelle, utilisez un [verrou de ressource][resource-lock] pour verrouiller tout le groupe de ressources ou des ressources individuelles, par exemple une machine virtuelle.

## <a name="security-considerations"></a>Considérations relatives à la sécurité

[Azure Security Center][security-center] vous offre un aperçu global de l’état de toutes vos ressources Azure en termes de sécurité. Il surveille les problèmes potentiels de sécurité et fournit une image complète de la sécurité de votre déploiement. Le Centre de sécurité est configuré pour chaque abonnement Azure. Activez la collecte de données de sécurité comme décrit dans le [Guide de démarrage rapide d’Azure Security Center][security-center-get-started]. Une fois la collecte de données activée, le Centre de sécurité analyse automatiquement les machines virtuelles créées dans le cadre de cet abonnement.

**Gestion des correctifs.** Si cette option est activée, Security Center vérifie si des mises à jour critiques et de sécurité sont manquantes. 

**Logiciel anti-programme malveillant.** Si cette option est activée, le Centre de sécurité vérifie si un logiciel anti-programme malveillant est installé. Vous pouvez également utiliser le Centre de sécurité pour installer des logiciels anti-programme malveillant dans le portail Azure.

**Opérations.** Utilisez le [contrôle d’accès en fonction du rôle][rbac] (RBAC) pour contrôler l’accès aux ressources Azure que vous déployez. Le contrôle RBAC vous permet d’affecter des rôles d’autorisation aux membres de votre équipe DevOps. Par exemple, le rôle Lecteur permet d’afficher des ressources Azure mais pas de les créer, gérer ou supprimer. Certains rôles sont spécifiques à des types de ressources Azure particuliers. Par exemple, le rôle Contributeur de machine virtuelle peut redémarrer ou désallouer une machine virtuelle, réinitialiser le mot de passe administrateur, créer une machine virtuelle, et ainsi de suite. D’autres [rôles RBAC intégrés][rbac-roles] peuvent être utiles dans cette architecture, notamment [Utilisateur DevTest Lab][rbac-devtest] et [Collaborateur de réseau][rbac-network]. Un utilisateur peut être affecté à plusieurs rôles, et vous pouvez créer des rôles personnalisés pour d’autres autorisations plus affinées.

> [!NOTE]
> Le contrôle RBAC ne limite pas les actions qu’un utilisateur connecté peut effectuer sur une machine virtuelle. Ces autorisations dépendent du type de compte installé sur le système d’exploitation invité.   

Utilisez les [journaux d’audit][audit-logs] pour voir les actions d’approvisionnement et d’autres événements concernant la machine virtuelle.

**Chiffrement des données.** Utilisez [Azure Disk Encryption][disk-encryption] si vous devez chiffrer les disques du système d’exploitation et de données. 

## <a name="deploy-the-solution"></a>Déployer la solution

Un déploiement pour cette architecture est disponible sur [GitHub][github-folder]. Il déploie les éléments suivants :

  * Un réseau virtuel avec un seul sous-réseau nommé **web** qui sert à héberger la machine virtuelle.
  * Un groupe de sécurité réseau avec deux règles de trafic entrant pour autoriser le trafic SSH et HTTP vers la machine virtuelle.
  * Une machine virtuelle exécutant la version la plus récente d’Ubuntu 16.04.3 LTS.
  * Un exemple d’extension de script personnalisé qui formate les deux disques de données et déploie Apache HTTP Server sur la machine virtuelle Ubuntu.

### <a name="prerequisites"></a>Composants requis

Avant de pouvoir déployer l’architecture de référence sur votre propre abonnement, vous devez effectuer les étapes suivantes.

1. Clonez, dupliquez ou téléchargez le fichier zip pour le dépôt GitHub des [architectures de référence AzureCAT][ref-arch-repo].

2. Vérifiez qu’Azure CLI 2.0 est installé sur votre ordinateur. Pour des instructions d’installation de l’interface de ligne de commande, consultez [Installer Azure CLI 2.0][azure-cli-2].

3. Installez le package npm des [modules Azure][azbb].

4. À partir d’une invite de commandes, d’une invite bash ou de l’invite de commandes PowerShell, connectez-vous à votre compte Azure à l’aide de l’une des commandes ci-dessous et suivez les invites.

  ```bash
  az login
  ```

### <a name="deploy-the-solution-using-azbb"></a>Déployer la solution à l’aide d’azbb

Pour déployer l’exemple de charge de travail de machine virtuelle unique, effectuez les étapes suivantes :

1. Accédez au dossier `virtual-machines\single-vm\parameters\linux` pour le dépôt que vous avez téléchargé à l’étape des prérequis ci-dessus.

2. Ouvrez le fichier `single-vm-v2.json` et entrez un nom d’utilisateur et une clé publique SSH entre les guillemets, comme illustré ci-dessous, puis enregistrez le fichier.

  ```bash
  "adminUsername": "",
  "sshPublicKey": "",
  ```

3. Exécutez `azbb` pour déployer l’exemple de machine virtuelle, comme illustré ci-dessous.

  ```bash
  azbb -s <subscription_id> -g <resource_group_name> -l <location> -p single-vm-v2.json --deploy
  ```

Pour plus d’informations sur le déploiement de cet exemple d’architecture de référence, visitez notre [dépôt GitHub][git].

## <a name="next-steps"></a>Étapes suivantes

- En savoir plus sur nos [modules Azure][azbbv2].
- Déployer [plusieurs machines virtuelles][multi-vm] dans Azure.

<!-- links -->
[audit-logs]: https://azure.microsoft.com/blog/analyze-azure-audit-logs-in-powerbi-more/
[availability-set]: /azure/virtual-machines/virtual-machines-linux-manage-availability
[azbb]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[azbbv2]: https://github.com/mspnp/template-building-blocks
[azure-cli-2]: /cli/azure/install-azure-cli?view=azure-cli-latest
[azure-linux]: /azure/virtual-machines/virtual-machines-linux-azure-overview
[azure-storage]: /azure/storage/storage-introduction
[blob-snapshot]: /azure/storage/storage-blob-snapshots
[blob-storage]: /azure/storage/storage-introduction
[boot-diagnostics]: https://azure.microsoft.com/blog/boot-diagnostics-for-virtual-machines-v2/
[cname-record]: https://en.wikipedia.org/wiki/CNAME_record
[data-disk]: /azure/virtual-machines/virtual-machines-linux-about-disks-vhds
[disk-encryption]: /azure/security/azure-security-disk-encryption
[enable-monitoring]: /azure/monitoring-and-diagnostics/insights-how-to-use-diagnostics
[fqdn]: /azure/virtual-machines/virtual-machines-linux-portal-create-fqdn
[git]: https://github.com/mspnp/reference-architectures/tree/master/virtual-machines/single-vm
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/virtual-machines/single-vm
[iostat]: https://en.wikipedia.org/wiki/Iostat
[manage-vm-availability]: /azure/virtual-machines/virtual-machines-linux-manage-availability
[multi-vm]: multi-vm.md
[naming-conventions]: /azure/architecture/best-practices/naming-conventions.md
[nsg]: /azure/virtual-network/virtual-networks-nsg
[nsg-default-rules]: /azure/virtual-network/virtual-networks-nsg#default-rules
[planned-maintenance]: /azure/virtual-machines/virtual-machines-linux-planned-maintenance
[premium-storage]: /azure/virtual-machines/linux/premium-storage
[premium-storage-supported]: /azure/virtual-machines/linux/premium-storage#supported-vms
[rbac]: /azure/active-directory/role-based-access-control-what-is
[rbac-roles]: /azure/active-directory/role-based-access-built-in-roles
[rbac-devtest]: /azure/active-directory/role-based-access-built-in-roles#devtest-labs-user
[rbac-network]: /azure/active-directory/role-based-access-built-in-roles#network-contributor
[reboot-logs]: https://azure.microsoft.com/blog/viewing-vm-reboot-logs/
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[resource-lock]: /azure/resource-group-lock-resources
[resource-manager-overview]: /azure/azure-resource-manager/resource-group-overview
[security-center]: /azure/security-center/security-center-intro
[security-center-get-started]: /azure/security-center/security-center-get-started
[select-vm-image]: /azure/virtual-machines/virtual-machines-linux-cli-ps-findimage
[services-by-region]: https://azure.microsoft.com/regions/#services
[ssh-linux]: /azure/virtual-machines/virtual-machines-linux-mac-create-ssh-keys
[static-ip]: /azure/virtual-network/virtual-networks-reserved-public-ip
[virtual-machine-sizes]: /azure/virtual-machines/virtual-machines-linux-sizes
[visio-download]: https://archcenter.azureedge.net/cdn/vm-reference-architectures.vsdx
[vm-disk-limits]: /azure/azure-subscription-service-limits#virtual-machine-disk-limits
[vm-resize]: /azure/virtual-machines/virtual-machines-linux-change-vm-size
[vm-size-tables]: /azure/virtual-machines/virtual-machines-linux-sizes
[vm-sla]: https://azure.microsoft.com/support/legal/sla/virtual-machines
[0]: ./images/single-vm-diagram.png "Architecture de machine virtuelle Linux unique dans Azure"