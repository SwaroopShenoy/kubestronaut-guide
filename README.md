# Kubestronaut Study Guide: The Five CNCF Kubernetes Certifications

*Five certifications, five sections, one path from first cluster to securing production.*

Kubestronaut is the title held by people who have earned all five CNCF Kubernetes certifications: KCNA, KCSA, CKA, CKAD and CKS. This guide takes each one in turn. Every section is organised by the official exam domains, and every topic explains the concepts, the commands and the reasoning behind them.

## About this guide

- **Independent.** This guide is not produced by, affiliated with, or endorsed by the Cloud Native Computing Foundation, The Linux Foundation, or the Kubernetes project.
- **Checked against official curricula.** Domain names and weights come from the official CNCF exam curriculum PDFs, checked on 2026-10-05. Each section has a *Sources and verification* page with the links and a list of what was not verified.
- **Facts change.** Exam formats, passing scores, allowed documentation and Kubernetes versions change over time. Check the official pages before you rely on any of them.
- **Test before you trust.** Commands and manifests are based on documented behaviour and have not all been run on a live cluster. Try them in a lab first.
- **Not a replacement.** This guide does not replace the official documentation or hands-on practice.
- **No warranty.** The author accepts no responsibility for exam results, for changes made to any system, or for decisions taken on the basis of this content.

## Contents

- [Section I: KCNA, Kubernetes and Cloud Native Associate](kcna/README.md)
- [Section II: KCSA, Kubernetes and Cloud Native Security Associate](kcsa/README.md)
- [Section III: CKA, Certified Kubernetes Administrator](cka/README.md)
- [Section IV: CKAD, Certified Kubernetes Application Developer](ckad/README.md)
- [Section V: CKS, Certified Kubernetes Security Specialist](cks/README.md)

## Raw source material

Each section has an `_archive/` folder with the unedited notes used during preparation. They are messy, repetitive and partly outdated, and they are not the guide's content. The cleaned topics are the reference; the archives are there for anyone who wants to see the raw material, with the warning that it can be a rabbit hole.

## Suggested paths

- **From the beginning:** KCNA, then KCSA, then CKA, then CKAD, then CKS. Each section builds on the ones before it.
- **Administrator track:** KCNA, CKA, then CKS.
- **Developer track:** KCNA, then CKAD.
- **Security track:** KCNA, KCSA, then CKS.

## Requirements to note

The CKS certification requires that you have passed the CKA at some point before you register. The CKA does not need to be active. This is stated on the CNCF CKS certification page.
