---
title: "Virtualisation — La classification Type 1 / Type 2 a une lacune"
date: 2026-05-31
draft: false
description: "La classification Type 1 / Type 2 des hyperviseurs a une lacune — KVM, Hyper-V et Hypervisor.framework n'entrent pas proprement dans l'une ou l'autre catégorie, et comprendre pourquoi compte plus que le label."
tags: ["virtualisation", "hyperviseur", "kvm", "docker", "conteneurs", "linux", "systèmes"]
---

La classification Type 1 / Type 2 des hyperviseurs a une lacune.

Le Type 1 démarre avant l'OS et accède directement au matériel — ESXi, Xen. Le Type 2 tourne au-dessus de l'OS et passe par lui — VirtualBox classique, VMware Workstation. La distinction est architecturale : où l'hyperviseur se situe par rapport à l'OS. Ce n'est pas une question d'émulation — les deux types utilisent les extensions de virtualisation matérielle quand elles sont disponibles.

La lacune, c'est KVM, Hyper-V et Hypervisor.framework d'Apple. Ils tournent dans le noyau OS, donc ils sont hébergés comme un Type 2. Mais le noyau n'est pas un intermédiaire ici — c'est lui l'hyperviseur. Ils accèdent directement aux extensions de virtualisation du CPU, comme un Type 1. KVM est dans le noyau Linux depuis 2007. Hyper-V est intégré à Windows. Hypervisor.framework fait partie de XNU. Ce ne sont pas des add-ons — c'est le noyau qui expose directement les instructions de virtualisation CPU aux applications qui ont besoin de créer des VMs.

VMware et VirtualBox sur les systèmes modernes utilisent ces mécanismes en sous-main. Les outils que les gens considèrent comme "Type 2" reposent eux-mêmes là-dessus.

Certains appellent ça "Type 1.5." Le label n'a pas d'importance. Ce qui compte, c'est que la classification est antérieure à ce modèle et ne le décrit pas bien.

Je l'ai rencontré concrètement : Docker sur un Mac M3 a besoin d'une VM Linux. Ce qui la crée, c'est Hypervisor.framework — intégré au noyau, accéléré matériellement, overhead minimal. Même histoire sous Linux avec KVM, même chose sous Windows avec Hyper-V. Un seul niveau de virtualisation, pas deux.

Les conteneurs, c'est autre chose. Pas de matériel émulé, pas de noyau isolé — juste des namespaces Linux et des cgroups. Un conteneur est un processus. Le noyau est entièrement partagé. C'est pour ça que les conteneurs démarrent en millisecondes et les VMs en minutes.

Le modèle enseigné dans la plupart des cours n'est pas faux. Il s'arrête juste avant cette partie.
