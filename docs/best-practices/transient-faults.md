---
title: "Conseils généraux sur les nouvelles tentatives"
description: Conseils sur les nouvelles tentatives dans le cadre de la gestion des erreurs temporaires.
author: dragon119
ms.date: 07/13/2016
pnp.series.title: Best Practices
ms.openlocfilehash: 9562e3447b2219fe2f3df96cfca24b845efa39b0
ms.sourcegitcommit: c53adf50d3a787956fc4ebc951b163a10eeb5d20
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 11/23/2017
---
# <a name="transient-fault-handling"></a>Gestion des erreurs temporaires

[!INCLUDE [header](../_includes/header.md)]

Toutes les applications qui communiquent avec des services à distance ou des ressources distantes doivent être sensibles aux erreurs temporaires. Cela est particulièrement vrai pour les applications exécutées dans le cloud, où la nature de l’environnement et la connectivité via Internet augmentent le risque de rencontrer ce type d’erreurs. Les erreurs temporaires incluent la perte momentanée de la connectivité réseau aux composants et aux services, l’indisponibilité passagère d’un service ou les expirations de délai qui surviennent lorsqu’un service est occupé. Ces erreurs se corrigent souvent d’elles-mêmes, et si l’action est répétée après un délai approprié, il est probable qu’elle aboutisse.

Ce document présente des conseils généraux pour la gestion des erreurs temporaires. Pour plus d’informations sur la gestion des erreurs temporaires dans le cadre de l’utilisation des services Microsoft Azure, consultez [Instructions relatives aux nouvelles tentatives pour les services Azure](./retry-service-specific.md).

## <a name="why-do-transient-faults-occur-in-the-cloud"></a>Pourquoi des erreurs temporaires se produisent-elles dans le cloud ?
Les erreurs temporaires peuvent se produire dans n’importe quel environnement, sur n’importe quel système d’exploitation ou plateforme et dans n’importe quel type d’application. Dans le cas de solutions exécutées sur une infrastructure locale, les performances et la disponibilité de l’application et de ses composants sont généralement garanties à travers une redondance matérielle coûteuse et souvent sous-exploitée, et les ressources et les composants sont situés à proximité les uns des autres. Bien que cette configuration réduise les risques de défaillance, elle peut toujours entraîner des erreurs temporaires, voire une panne suite à des événements imprévus tels que des problèmes d’alimentation externe ou de réseau, ou d’autres scénarios menant à l’arrêt de l’activité.

L’hébergement cloud, y compris les systèmes de cloud privé, peut offrir une disponibilité globale supérieure en s’appuyant sur des ressources partagées, la redondance, le basculement automatique et l’allocation dynamique des ressources entre un très grand nombre de nœuds de calcul de base. Cependant, la nature de ces environnements est susceptible d’accroître le risque de survenue d’erreurs temporaires. Il existe plusieurs raisons à cela :

* Dans un environnement de cloud, beaucoup de ressources sont partagées et l’accès à ces ressources fait l’objet d’une limitation afin de protéger ces dernières. Certains services refusent les connexions lorsque la charge s’élève à un niveau spécifique ou qu’un débit maximal est atteint afin de permettre le traitement des demandes existantes et de garantir un niveau de performances optimal du service pour tous les utilisateurs. Cette limitation contribue à maintenir la qualité de service pour les voisins et les autres clients utilisant les ressources partagées.
* Les environnements de cloud sont créés à l’aide de très nombreuses unités matérielles de base. Ils doivent leurs performances à la répartition dynamique de la charge entre plusieurs unités de calcul et composants d’infrastructure et leur fiabilité au recyclage ou au remplacement automatique des unités défaillantes. Cette nature dynamique peut entraîner occasionnellement des échecs de connexion et des erreurs temporaires.
* Il y a souvent plus de composants matériels, notamment des composants d’infrastructure réseau tels que des routeurs et des équilibreurs de charge, entre l’application et les ressources et services qu’elle utilise. Cette infrastructure en plus peut parfois introduire une latence et des erreurs de connexion temporaires supplémentaires.
* Les conditions réseau entre le client et le serveur peuvent être variables, tout particulièrement lorsque les communications franchissent Internet. Même dans des emplacements locaux, les charges de trafic très importantes peuvent ralentir les communications et entraîner des échecs de connexion intermittents.

## <a name="challenges"></a>Défis
Les erreurs temporaires peuvent avoir un impact considérable sur la perception de la disponibilité d’une application, même si cette dernière a été testée de manière approfondie dans toutes les circonstances prévisibles. Pour fonctionner de façon fiable, les applications hébergées sur le cloud doivent être capables de répondre aux défis suivants :

* L’application doit être en mesure de détecter les erreurs lorsqu’elles se produisent et de déterminer si ces erreurs sont susceptibles d’être temporaires, de plus longue durée ou s’il s’agit de défaillances irréversibles. Les différentes ressources renvoient généralement des réponses différentes lorsqu’une erreur se produit, et ces réponses peuvent également varier selon le contexte de l’opération ; par exemple, la réponse à une erreur survenant lors de l’accès en lecture au stockage peut différer de la réponse à une erreur survenant lors de l’écriture dans le stockage. Des contrats bien documentés concernant les erreurs temporaires sont disponibles pour un grand nombre de ressources et de services. Cependant, en l’absence de telles informations, il peut s’avérer difficile de découvrir la nature de l’erreur et si cette dernière est susceptible d’être temporaire.
* Si l’application détermine que l’erreur est susceptible d’être temporaire, elle doit être en mesure de retenter l’opération et de suivre le nombre de tentatives effectuées.
* L’application doit s’appuyer sur une stratégie appropriée pour les nouvelles tentatives. Cette stratégie spécifie le nombre de nouvelles tentatives requises, le délai entre chaque tentative et les mesures à prendre après une tentative infructueuse. Le nombre approprié de tentatives et le délai entre chacune d’elle sont souvent difficiles à déterminer et varient en fonction du type de ressource, mais également des conditions de fonctionnement actuelles de la ressource et de l’application elle-même.

## <a name="general-guidelines"></a>Recommandations générales
Les recommandations suivantes vous aideront à concevoir un mécanisme de gestion des erreurs temporaires approprié pour vos applications :

* **Déterminez s’il existe un mécanisme de nouvelle tentative intégré :**
  * De nombreux services fournissent un kit de développement logiciel (SDK) ou une bibliothèque cliente contenant un mécanisme de gestion des erreurs temporaires intégré. La politique de nouvelle tentative utilisée est généralement adaptée à la nature et aux exigences du service cible. Les interfaces REST pour les services peuvent également renvoyer des informations utiles pour déterminer si une nouvelle tentative est appropriée et combien de temps attendre avant la tentative suivante.
  * Le cas échéant, utilisez le mécanisme de nouvelle tentative intégré, sauf si vous avez des besoins spécifiques et bien définis pour lesquels un comportement de nouvelle tentative différent serait plus approprié.
* **Déterminez si l’opération est adaptée à de nouvelles tentatives**:
  * Vous devez uniquement retenter les opérations où les erreurs sont temporaires (ce que révèle généralement la nature de l’erreur) et si l’opération a des chances d’aboutir en cas de nouvelle tentative. Il est inutile de retenter des opérations qui indiquent une opération non valide, telle qu’une mise à jour de base de données pour un élément inexistant ou les demandes adressées à un service ou une ressource qui a subi une erreur irrécupérable.
  * En général, vous devez implémenter les nouvelles tentatives uniquement lorsque vous pouvez en déterminer le plein impact et que les conditions sont bien comprises et peuvent être validées. Si ce n’est pas le cas, laissez le code appelant implémenter les nouvelles tentatives. Gardez à l’esprit que les erreurs renvoyées par des ressources et des services que vous ne contrôlez pas peuvent évoluer au fil du temps, et qu’il vous faudra peut-être revoir votre logique de détection des erreurs temporaires.
  * Lorsque vous créez des services ou des composants, envisagez d’implémenter des codes d’erreur et des messages qui aideront les clients à déterminer s’ils doivent retenter des opérations ayant échoué. Plus particulièrement, indiquez si le client doit retenter l’opération (par exemple en renvoyant une valeur **isTransient** ) et suggérez un délai approprié avant la tentative suivante. Si vous créez un service web, envisagez de renvoyer des erreurs personnalisées définies dans les contrats de service. Même s’il se peut que les clients génériques ne soient pas en mesure de les lire, elles seront utiles pour la création de clients personnalisés.
* **Déterminez un nombre de nouvelles tentatives et un intervalle appropriés :**
  * Il est essentiel d’optimiser le nombre de nouvelles tentatives et l’intervalle pour le type de cas d’utilisation. Si le nombre de nouvelles tentatives est insuffisant, l’application ne pourra pas terminer l’opération et risque de connaître une défaillance. Si le nombre de nouvelles tentatives est excessif ou l’intervalle entre elles trop court, l’application est susceptible de bloquer les ressources telles que les threads, les connexions et la mémoire pendant de longues périodes, ce qui nuira à l’intégrité de l’application.
  * Les valeurs appropriées pour l’intervalle de temps et le nombre de nouvelles tentatives dépendent du type d’opération concerné. Par exemple, si l’opération fait partie d’une interaction utilisateur, l’intervalle doit être court et quelques nouvelles tentatives seulement doivent être effectuées pour éviter de faire attendre une réponse aux utilisateurs (ce qui maintient les connexions ouvertes et peut réduire la disponibilité pour les autres utilisateurs). Si l’opération fait partie d’un workflow de longue durée ou critique, pour lequel il serait onéreux ou fastidieux d’annuler et de redémarrer le processus, il convient d’attendre plus longtemps entre les tentatives et d’effectuer un plus grand nombre de nouvelles tentatives.
  * Déterminer l’intervalle approprié entre les nouvelles tentatives est l’étape la plus difficile de la conception d’une stratégie efficace. Les stratégies classiques utilisent les types suivants d’intervalles avant nouvelle tentative :
    * **Temporisation exponentielle**. L’application attend un court délai avant la première nouvelle tentative, puis augmente l’intervalle de façon exponentielle entre chaque tentative suivante. Par exemple, elle peut retenter l’opération après 3 secondes, 12 secondes, 30 secondes et ainsi de suite.
    * **Intervalles incrémentiels**. L’application attend un court délai avant la première nouvelle tentative, puis augmente l’intervalle de façon incrémentielle entre chaque tentative suivante. Par exemple, elle peut retenter l’opération après 3 secondes, 7 secondes, 13 secondes et ainsi de suite.
    * **Intervalles réguliers**. L’application attend le même laps de temps entre chaque tentative. Par exemple, elle peut retenter l’opération toutes les 3 secondes.
    * **Nouvelle tentative immédiate**. Une erreur temporaire peut être très courte, par exemple lorsqu’elle est causée par un événement tel qu’une collision de paquets réseau ou un pic dans un composant matériel. Dans ce cas, il est opportun de retenter l’opération immédiatement, car celle-ci peut aboutir si l’erreur a disparu le temps que l’application assemble et envoie la demande suivante. Cependant, il ne doit jamais y avoir plus d’une nouvelle tentative immédiate, et vous devez implémenter d’autres stratégies, telles que la temporisation exponentielle ou les actions de secours, en cas d’échec de la nouvelle tentative immédiate.
    * **Répartition aléatoire**. Toutes les stratégies de nouvelle tentative répertoriées ci-dessus peuvent inclure une répartition aléatoire pour éviter que plusieurs instances du client effectuent les tentatives suivantes en même temps. Par exemple, une instance peut retenter l’opération après 3 secondes, 11 secondes, 28 secondes et ainsi de suite, tandis qu’une autre instance peut retenter l’opération après 4 secondes, 12 secondes, 26 secondes et ainsi de suite. La répartition aléatoire est une technique utile qui peut être associée à d’autres stratégies.  
  * En règle générale, utilisez une stratégie de temporisation exponentielle pour les opérations d’arrière-plan et des stratégies de nouvelle tentative immédiate ou à intervalles réguliers pour les opérations interactives. Dans les deux cas, vous devez choisir le délai et le nombre de nouvelles tentatives de sorte que la latence maximale pour toutes les nouvelles tentatives soit comprise dans la plage de latence de bout en bout requise.
  * Prenez en compte la combinaison de tous les facteurs qui contribuent au délai d’expiration maximal global pour une opération retentée. Ces facteurs incluent le temps mis par une connexion ayant échoué pour produire une réponse (généralement défini par une valeur de délai d’expiration dans le client), ainsi que le délai entre les nouvelles tentatives et le nombre maximal de nouvelles tentatives. La somme de tous ces délais peut entraîner des durées d’opération globales très importantes, en particulier lors de l’utilisation d’une stratégie de délai exponentiel où l’intervalle entre les nouvelles tentatives augmente rapidement après chaque échec. Si un processus doit satisfaire un contrat de niveau de service (SLA) spécifique, la durée d’opération globale, y compris tous les délais d’expiration et autres, doit être comprise dans la plage définie dans le contrat SLA.
  * Les stratégies de nouvelle tentative trop agressives, qui présentent des intervalles trop courts ou un trop grand nombre de nouvelles tentatives, peuvent avoir un impact négatif sur la ressource ou le service cible. Cela peut empêcher la ressource ou le service de récupérer de son état de surcharge, et les demandes continueront à être bloquées ou rejetées. Il en résulte un cercle vicieux où des demandes toujours plus nombreuses sont envoyées à la ressource ou au service, réduisant de plus en plus sa capacité de récupération.
  * Prenez en compte le délai d’expiration des opérations lorsque vous choisissez les intervalles pour éviter de lancer une nouvelle tentative immédiatement (par exemple, si le délai d’expiration est similaire à l’intervalle avant nouvelle tentative). Demandez-vous également si vous devez maintenir la période totale possible (le délai d’expiration plus les intervalles avant nouvelle tentative) en dessous d’une durée totale spécifique. Les opérations ayant des délais d’expiration exceptionnellement courts ou très longs peuvent avoir une influence sur le délai d’attente et la fréquence à laquelle retenter l’opération.
  * Utilisez le type de l’exception et les données qu’il contient ou les codes d’erreur et les messages renvoyés par le service afin d’optimiser l’intervalle et le nombre de nouvelles tentatives. Par exemple, un certain nombre d’exceptions ou de codes d’erreur (tels que le code HTTP 503 Service indisponible avec un en-tête Retry-After dans la réponse) peuvent indiquer la durée potentielle de l’erreur ou que le service a échoué et ne répondra pas aux nouvelles tentatives.
* **Évitez les anti-modèles**:
  * Dans la plupart des cas, vous devez éviter des implémentations qui incluent des couches dupliquées de code de nouvelle tentative. Évitez les conceptions qui incluent des mécanismes de nouvelle tentative en cascade ou qui implémentent la nouvelle tentative à chaque étape d’une opération impliquant une hiérarchie de demandes, sauf si vous avez des besoins spécifiques qui l’exigent. Dans ces circonstances exceptionnelles, utilisez des politiques qui empêchent un nombre de nouvelles tentatives et des délais excessifs, et assurez-vous que vous comprenez les répercussions. Par exemple, si un composant effectue une demande à un autre composant, qui accède ensuite au service cible, et que vous implémentez les nouvelles tentatives au nombre de trois pour les deux appels, il y aura neuf tentatives au total pour le service. Un grand nombre de services et de ressources implémentent un mécanisme de nouvelle tentative intégré et vous devez examiner comment désactiver ou modifier celui-ci si vous avez besoin d’implémenter de nouvelles tentatives à un niveau supérieur.
  * N’implémentez jamais de mécanisme de nouvelle tentative infini. Vous risqueriez d’empêcher la ressource ou le service de récupérer des situations de surcharge, et de prolonger la limitation et le rejet des connexions. Utilisez un nombre fini de nouvelles tentatives ou implémentez un modèle tel que [Disjoncteur](http://msdn.microsoft.com/library/dn589784.aspx) pour permettre au service de récupérer.
  * N’effectuez jamais plus d’une nouvelle tentative immédiate.
  * Évitez d’utiliser un intervalle avant nouvelle tentative régulier, en particulier avec un nombre important de nouvelles tentatives, pour l’accès aux services et aux ressources dans Azure. L’approche optimale pour ce scénario consiste à utiliser une stratégie de temporisation exponentielle avec une capacité de disjoncteur.
  * Empêchez plusieurs instances du même client ou plusieurs instances de clients différents d’effectuer de nouvelles tentatives au même moment. Si cela risque de se produire, introduisez la répartition aléatoire dans les intervalles avant nouvelle tentative.
* **Testez votre stratégie de nouvelle tentative et son implémentation :**
  * Veillez à tester entièrement l’implémentation de votre stratégie de nouvelle tentative dans un éventail de circonstances aussi large que possible, en particulier lorsque l’application et les ressources ou services cibles qu’elle utilise sont soumis à une charge extrême. Pour vérifier le comportement pendant le test, vous pouvez :
    * Injecter des erreurs temporaires et non temporaires dans le service. Par exemple, envoyez des demandes non valides ou ajoutez du code qui détecte les requêtes test et répond avec différents types d’erreurs. Pour un exemple d’utilisation de TestApi, consultez les articles [Test d’injection d’erreurs avec TestApi](http://msdn.microsoft.com/magazine/ff898404.aspx) et [Présentation de TestApi - Partie 5 : API d’injection d’erreurs dans du code managé](http://blogs.msdn.com/b/ivo_manolov/archive/2009/11/25/9928447.aspx).
    * Créer une ressource ou un service fictif renvoyant une plage d’erreurs que le service réel peut renvoyer. Veillez à couvrir tous les types d’erreurs pour la détection desquels votre stratégie de nouvelle tentative est conçue.
    * Forcer la survenue d’erreurs temporaires en désactivant ou en surchargeant temporairement le service s’il s’agit d’un service personnalisé que vous avez créé et déployé (vous ne devez pas, bien entendu, essayer de surcharger les ressources ou services partagés dans Azure).
    * Pour les API HTTP, envisagez d’utiliser la bibliothèque FiddlerCore dans vos tests automatisés pour modifier le résultat des demandes HTTP, soit en ajoutant des temps d’aller-retour supplémentaires, soit en modifiant la réponse (par exemple, le code d’état, les en-têtes et le corps HTTP ou d’autres facteurs). Cela permet de tester de manière déterministe un sous-ensemble de conditions d’échec, qu’il s’agisse d’erreurs temporaires ou d’autres types de défaillances. Pour plus d’informations, consultez la page [FiddlerCore](http://www.telerik.com/fiddler/fiddlercore). Pour obtenir des exemples d’utilisation de la bibliothèque, en particulier la classe **HttpMangler** , examinez le [code source pour le Kit de développement logiciel (SDK) Azure Storage](https://github.com/Azure/azure-storage-net/tree/master/Test).
    * Effectuer des tests de facteur de charge élevé et des tests simultanés pour vous assurer que le mécanisme et la stratégie de nouvelle tentative fonctionnent correctement dans ces conditions et n’ont pas un impact négatif sur le fonctionnement du client ou n’entraînent pas de contamination croisée entre les demandes.
* **Gérez les configurations de politique de nouvelle tentative :**
  * Une *politique de nouvelle tentative* associe tous les éléments de votre stratégie de nouvelle tentative. Elle définit le mécanisme de détection qui détermine si une erreur est susceptible d’être temporaire, le type d’intervalle à utiliser (régulier, temporisation exponentielle, répartition aléatoire, etc.), les valeurs d’intervalle réelles et le nombre de nouvelles tentatives à effectuer.
  * Les nouvelles tentatives doivent être implémentées à de nombreux endroits, même pour la plus simple des applications, et dans chaque couche des applications plus complexes. Au lieu de coder en dur les éléments de chaque politique à plusieurs emplacements, envisagez d’utiliser un point central pour stocker l’ensemble des politiques. Par exemple, stockez les valeurs telles que l’intervalle et le nombre de tentatives dans les fichiers de configuration de l’application, lisez ces derniers à l’exécution et créez les politiques de nouvelle tentative par programme. Cela facilite la gestion des paramètres, ainsi que la modification et l’ajustement des valeurs afin de répondre à l’évolution des besoins et des scénarios. Cependant, concevez le système de sorte qu’il stocke les valeurs plutôt que de relire un fichier de configuration à chaque fois et assurez-vous que des valeurs par défaut appropriées sont utilisées si les valeurs ne peuvent pas être obtenues à partir de la configuration.
  * Dans une application Azure Cloud Services, envisagez de stocker les valeurs qui sont utilisées pour créer les politiques de nouvelle tentative à l’exécution dans le fichier de configuration de service afin qu’elles puissent être modifiées sans avoir à redémarrer l’application.
  * Tirez parti des stratégies de nouvelle tentative intégrées ou par défaut disponibles dans les API clientes que vous utilisez, mais uniquement lorsqu’elles sont adaptées à votre scénario. Ces stratégies sont généralement conçues pour un usage général. Elles peuvent suffire à satisfaire l’ensemble des exigences de certains scénarios, mais pour d’autres, il se peut qu’elles n’offrent pas toutes les options nécessaires pour répondre à vos besoins spécifiques. Vous devez comprendre comment les paramètres affecteront votre application à l’aide de tests afin de déterminer les valeurs les plus appropriées.
* **Consignez et suivez les erreurs temporaires et non temporaires :**
  * Dans le cadre de votre stratégie de nouvelle tentative, incluez la gestion des exceptions et d’autres systèmes d’instrumentation permettant de consigner chaque nouvelle tentative. Bien que des erreurs temporaires et de nouvelles tentatives occasionnelles soient à prévoir et ne soient pas symptomatiques d’un problème, de nouvelles tentatives régulières et de plus en plus nombreuses sont souvent le signe d’un problème pouvant entraîner une défaillance ou affectant actuellement les performances et la disponibilité de l’application.
  * Consignez les erreurs temporaires en tant qu’avertissements plutôt qu’en tant qu’erreurs afin que les systèmes de surveillance ne les détectent pas comme des erreurs d’application susceptibles de déclencher de fausses alertes.
  * Envisagez de stocker dans vos entrées de journal une valeur qui indique si les nouvelles tentatives ont été provoquées par une limitation dans le service ou par d’autres types d’erreurs, tels que des échecs de connexion, afin de pouvoir les distinguer lors de l’analyse des données. Une augmentation du nombre d’erreurs de limitation est souvent révélatrice d’un défaut de conception de l’application ou de la nécessité de passer à un service Premium qui offre du matériel dédié.  
  * Envisagez de mesurer et de consigner le temps total nécessaire aux opérations qui incluent un mécanisme de nouvelle tentative. C’est un bon indicateur de l’effet global des erreurs temporaires sur les temps de réponse pour l’utilisateur, la latence des processus et l’efficacité des cas d’utilisation de l’application. Consignez également le nombre de tentatives effectuées pour comprendre les facteurs qui ont contribué au temps de réponse.
  * Envisagez d’implémenter un système de télémétrie et de surveillance capable de déclencher des alertes lorsque le nombre et le taux de défaillances, le nombre moyen de nouvelles tentatives ou le temps total que mettent les opérations pour aboutir augmentent.
* **Gérez les opérations qui échouent continuellement :**
  
  * Dans certains cas, il se peut que l’opération échoue à chaque tentative, et il est indispensable de réfléchir à la façon dont vous gérerez cette situation :
    * Bien qu’une stratégie de nouvelle tentative définisse le nombre maximal de fois qu’une opération doit être retentée, elle n’empêche pas ensuite l’application de répéter l’opération avec le même nombre de nouvelles tentatives. Par exemple, si un service de traitement des commandes échoue avec une erreur irrécupérable qui le met définitivement en défaut, la stratégie de nouvelle tentative peut détecter une expiration du délai de connexion et considérer cet événement comme une erreur temporaire. Le code retentera l’opération un nombre spécifié de fois avant d’abandonner. Cependant, lorsqu’un autre client passe une commande, l’opération sera tentée à nouveau, même si elle est vouée à l’échec à chaque fois.
    * Pour empêcher la poursuite des nouvelles tentatives pour les opérations qui échouent continuellement, envisagez d’implémenter le [modèle Disjoncteur](http://msdn.microsoft.com/library/dn589784.aspx). Dans ce modèle, si le nombre d’échecs dans un délai spécifié dépasse la limite autorisée, les demandes sont immédiatement renvoyées à l’appelant comme des erreurs, sans tenter d’accéder à la ressource ou au service qui a échoué.
    * L’application peut tester régulièrement le service, par intermittence et avec de très longs intervalles entre les demandes, pour détecter le moment où il redevient disponible. L’intervalle approprié dépend du scénario, par exemple de l’importance de l’opération et de la nature du service, et peut être compris entre quelques minutes et plusieurs heures. Lorsque le test réussit, l’application peut reprendre un fonctionnement normal et transmettre les demandes au service qui vient d’être restauré.
    * En attendant, vous pouvez avoir recours à une autre instance du service (par exemple dans un autre centre de données ou une autre application), utiliser un service similaire qui offre des fonctionnalités compatibles (peut-être plus simples) ou effectuer d’autres opérations en espérant que le service sera de nouveau disponible bientôt. Par exemple, il peut être judicieux de stocker les demandes liées au service dans une file d’attente ou un magasin de données et de les relire ultérieurement. Sinon, vous pourrez peut-être rediriger l’utilisateur vers une autre instance de l’application, dégrader les performances de l’application tout en continuant à offrir des fonctionnalités acceptables ou renvoyer simplement un message à l’utilisateur indiquant que l’application est actuellement indisponible.
* **Autres points à considérer**
  
  * Lorsque vous choisissez les valeurs pour le nombre de nouvelles tentatives et les intervalles avant nouvelle tentative d’une politique, demandez-vous si l’opération à effectuer sur le service ou la ressource fait partie d’une opération de longue durée ou à plusieurs étapes. Lorsqu’une étape opérationnelle échoue, il peut être difficile ou coûteux de compenser toutes les autres ayant déjà abouti. Dans ce cas, un intervalle très long et un grand nombre de nouvelles tentatives peuvent être acceptables à condition que les autres opérations ne soient pas bloquées par la mise en attente ou le verrouillage de ressources rares.
  * Demandez-vous si une nouvelle tentative pour la même opération peut donner lieu à des incohérences dans les données. Si certaines parties d’un processus à plusieurs étapes sont répétées et que les opérations ne sont pas idempotentes, cela risque d’entraîner une incohérence. Par exemple, si une opération qui incrémente une valeur est répétée, elle produira un résultat non valide. La répétition d’une opération qui envoie un message à une file d’attente peut entraîner une incohérence dans le consommateur de message si celui-ci ne prend pas en charge la détection des messages en double. Pour éviter ce problème, veillez à concevoir chaque étape comme une opération idempotente. Pour plus d’informations sur l’idempotence, consultez l’article [Modèles d’idempotence][idempotency-patterns].
  * Réfléchissez à la portée des opérations qui vont être retentées. Par exemple, il peut s’avérer plus simple d’implémenter du code de nouvelle tentative à un niveau qui englobe plusieurs opérations et de toutes les retenter en cas de défaillance de l’une d’elles. Cependant, cela peut donner lieu à des problèmes d’idempotence ou à des opérations de restauration inutiles.
  * Si vous choisissez une portée de nouvelle tentative qui englobe plusieurs opérations, prenez en compte la latence totale de toutes ces opérations pour déterminer les intervalles avant nouvelle tentative et surveiller le temps nécessaire, ou avant de déclencher des alertes pour des défaillances.
  * Réfléchissez à l’impact que votre stratégie de nouvelle tentative peut avoir sur les voisins et les autres clients au sein d’une application partagée ou lors de l’utilisation de ressources et de services partagés. Les politiques de nouvelle tentative agressives peuvent augmenter le nombre d’erreurs temporaires pour ces autres utilisateurs et pour les applications qui partagent les ressources et les services. De même, votre application peut être affectée par les politiques de nouvelle tentative implémentées par les autres utilisateurs des ressources et des services. Pour les applications stratégiques, vous pouvez décider d’utiliser des services Premium qui ne sont pas partagés. Vous bénéficierez ainsi d’un contrôle considérablement accru sur la charge de ces ressources et services et sur la limitation qui en découle, ce qui peut contribuer à justifier le coût supplémentaire de ces services.

## <a name="more-information"></a>Plus d’informations
* [Instructions relatives aux nouvelles tentatives pour les services Azure](./retry-service-specific.md)
* [Bloc applicatif de gestion des erreurs temporaires](http://msdn.microsoft.com/library/hh680934.aspx)
* [Modèle Disjoncteur](http://msdn.microsoft.com/library/dn589784.aspx)
* [Modèle de transaction de compensation](http://msdn.microsoft.com/library/dn589804.aspx)
* [Modèles d’idempotence][idempotency-patterns]

[idempotency-patterns]: http://blog.jonathanoliver.com/idempotency-patterns/
