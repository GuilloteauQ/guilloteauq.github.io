---
title: Thesis Defense
permalink: /defense/
---


I am pleased to invite you to the defense of my Ph.D. thesis entitled:

    "Control-based runtime management of HPC systems with support for reproducible experiments"

[Current version of the manuscript](https://cloud.univ-grenoble-alpes.fr/s/bmdzdzm92w4jzq8)

# Date and place:

**December 11th, 10am CET**, at the Amphitheater of the Bâtiment IMAG, 150 place du Torrent, 38401 Saint Martin d'Hères

**It will be possible to attend remotely.
The visio link is not set yet but will be updated here.**

# Jury:

- Alexandru COSTAN (INSA Rennes, IRISA) - Maître de conférence, HDR - Rapporteur
- Alessandro PAPADOPOULOS (Mälardalen University) - Professeur - Rapporteur
- Fabienne BOYER (Université Grenoble Alpes) - Maîtresse de conférence, HDR - Examinatrice
- Georges DA COSTA (Université Paul Sabatier) - Professeur - Examinateur
- Noël DE PALMA (Université Grenoble Alpes) - Professeur - Examinateur
- Eric RUTTEN (Inria Grenoble) - Chargé de recherche, HDR - Directeur de thèse
- Olivier RICHARD (Université Grenoble Alpes) - Maître de conférence - Co-encadrant de thèse


# Abstract: 

High-Performance Computing (HPC) systems have become increasingly more complex, and their performance and power consumption make them less predictable.
This unpredictability requires cautious runtime management to guarantee an acceptable Quality-of-Service to the end users.
Such a regulation problem arises in the context of the computing grid middleware CiGri that aims at harvesting the idle computing resources of a set of cluster by injection low priority jobs.
A too aggressive harvesting strategy can lead to the degradation of the performance for all the users of the clusters, while a too shy harvesting will leave resources idle and thus lose computing power.
There is thus a tradeoff between the amount of resources that can be harvested and the resulting degradation of users jobs, which can evolve at runtime based on Service Level Agreements and the current load of the system.

We claim that such regulation challenges can be addressed with tools from Autonomic Computing, and in particular when coupled with Control Theory.
This thesis investigates several regulation problems in the context of CiGri with such tools.
We will focus on regulating the harvesting based on the load of a shared distributed file-system, and improving the overall usage of the computing resources.
We will also evaluate and compare the reusability of the proposed control-based solutions in the context of HPC systems.

The experiments done in this thesis also led us to investigate new tools and techniques to improve the cost and reproducibility of the experiments.
We will present a tool named NixOS-compose able to generate and deploy reproducible distributed software environments.
We will also investigate techniques to reduce the number of machines needed to deploy experiments on grid or cluster middlewares, such as CiGri, while ensuring an acceptable level of realism for the final deployed system.
