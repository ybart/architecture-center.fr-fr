---
title: Utiliser les services gérés
description: Dans la mesure du possible, privilégiez l’approche platform as a service (PaaS) plutôt que l’approche infrastructure as a service (IaaS)
author: MikeWasson
layout: LandingPage
ms.openlocfilehash: 7156c073db3e047fb38e031309ddb637a9e44c02
ms.sourcegitcommit: b0482d49aab0526be386837702e7724c61232c60
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 11/14/2017
---
# <a name="use-managed-services"></a>Utiliser les services gérés

## <a name="when-possible-use-platform-as-a-service-paas-rather-than-infrastructure-as-a-service-iaas"></a>Dans la mesure du possible, privilégiez l’approche platform as a service (PaaS) plutôt que l’approche infrastructure as a service (IaaS)

L’approche IaaS équivaut à disposer d’un carton rempli de pièces. Vous pouvez construire tout ce que vous voulez, mais vous devez tout assembler vous-même. Les services gérés sont plus faciles à configurer et à administrer. Vous n’avez pas besoin d’approvisionner de machines virtuelles, de gérer les correctifs et les mises à jour, ni de prendre en charge les autres surcoûts inhérents à l’exécution de logiciels sur une machine virtuelle.

Par exemple, supposons que votre application ait besoin d’une file d’attente de messages. Vous pouvez configurer votre propre service de messagerie sur une machine virtuelle à l’aide d’un logiciel tel que RabbitMQ. Toutefois, Azure Service Bus fournit déjà une messagerie fiable sous la forme d’un service, et se révèle plus simple à configurer. Il vous suffit de créer un espace de noms Service Bus (ce qui peut être effectué dans le cadre d’un script de déploiement), puis d’appeler Service Bus à l’aide du Kit de développement logiciel (SDK) client. 

Bien entendu, il est possible que votre application présente certaines exigences auxquelles une approche IaaS peut répondre de façon plus adéquate. Toutefois, même si votre application repose sur IaaS, recherchez les emplacements où vous pouvez incorporer naturellement des services gérés. Ces emplacements comprennent le cache, les files d’attente et le stockage des données.

| Au lieu d’exécuter... | Utilisez plutôt... |
|-----------------------|-------------|
| Active Directory | Azure Active Directory Domain Services |
| Elasticsearch | Recherche Azure |
| Hadoop | HDInsight |
| IIS | App Service |
| MongoDB | Cosmos DB |
| Redis | Cache Redis Azure |
| SQL Server | Base de données SQL Azure |


