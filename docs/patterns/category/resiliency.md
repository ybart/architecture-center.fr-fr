---
title: "Modèles de résilience"
description: "La résilience est la capacité d’un système à traiter de manière appropriée les défaillances et à récupérer après ces dernières. La nature de l’hébergement cloud, où les applications sont souvent multi-locataires, utilisent les services de plateforme partagée, sont en concurrence pour la bande passante et les ressources, communiquent via Internet et s’exécutent sur du matériel de base, signifie qu’il existe une probabilité accrue que des erreurs temporaires et des erreurs plus permanentes se produisent. Pouvoir détecter les pannes et récupérer rapidement et efficacement est nécessaire afin de maintenir la résilience."
keywords: "modèle de conception"
author: dragon119
ms.date: 06/23/2017
pnp.series.title: Cloud Design Patterns
ms.openlocfilehash: a3b9d72989e0de57c689bcec51e20653d0441d31
ms.sourcegitcommit: b0482d49aab0526be386837702e7724c61232c60
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 11/14/2017
---
# <a name="resiliency-patterns"></a>Modèles de résilience

La résilience est la capacité d’un système à traiter de manière appropriée les défaillances et à récupérer après ces dernières. La nature de l’hébergement cloud, où les applications sont souvent multi-locataires, utilisent les services de plateforme partagée, sont en concurrence pour la bande passante et les ressources, communiquent via Internet et s’exécutent sur du matériel de base, signifie qu’il existe une probabilité accrue que des erreurs temporaires et des erreurs plus permanentes se produisent. Pouvoir détecter les pannes et récupérer rapidement et efficacement est nécessaire afin de maintenir la résilience.

| Modèle | Résumé |
| ------- | ------- |
| [Cloisonnement](../bulkhead.md) | Isolez les éléments d’une application sous forme de pools afin qu’en cas de défaillance de l’un d’eux, les autres continuent à fonctionner. |
| [Disjoncteur](../circuit-breaker.md) | Gérer les erreurs dont la résolution peut prendre un certain temps lors de la connexion à une ressource ou à un service distant. |
| [Transaction de compensation](../compensating-transaction.md) | Annulez le travail effectué par une série d’étapes qui définissent ensemble une opération cohérente. |
| [Surveillance de point de terminaison d’intégrité](../health-endpoint-monitoring.md) | Implémentez des contrôles fonctionnels dans une application à laquelle des outils externes peuvent accéder par le biais de points de terminaison exposés à intervalles réguliers. |
| [Élection du responsable](../leader-election.md) | Coordonnez les actions effectuées par un ensemble d’instances de tâche de collaboration dans une application distribuée en élisant l’instance responsable qui sera chargée de gérer les autres instances. |
| [Nivellement de la charge basé sur une file d’attente](../queue-based-load-leveling.md) | Utilisez une file d’attente qui agit comme mémoire tampon entre une tâche et un service qu’elle appelle, afin d’atténuer les surcharges intermittentes. |
| [Nouvelle tentative](../retry.md) | Permettez à une application de gérer les défaillances temporaires anticipées quand elle tente de se connecter à un service ou à une ressource réseau en réessayant d’exécuter en toute transparence une opération qui a échoué précédemment. |
| [Superviseur de l’agent du planificateur](../scheduler-agent-supervisor.md) | Coordonnez un ensemble d’actions sur un ensemble distribué de services et d’autres ressources à distance. |