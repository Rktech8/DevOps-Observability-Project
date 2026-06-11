
# Chapter 1: Introduction

## 1.1 Background

Modern software systems demand high availability and rapid fault recovery.
Traditional monitoring approaches rely on manual inspection, leading to
delayed detection and prolonged downtime. This project investigates whether
cloud-native observability tools can measurably reduce MTTR in a DevOps pipeline.

## 1.2 Problem Statement

When a containerized service fails in a production-like AWS environment,
how quickly can automated observability tooling detect and alert on the failure,
and what is the resulting Mean Time To Recovery (MTTR)?

## 1.3 Objectives

1. Build a containerized Nginx application deployed via GitHub Actions to AWS EC2
2. Integrate AWS CloudWatch for real-time log and metric collection
3. Configure SNS email alerts triggered by CloudWatch alarms
4. Simulate 5 controlled failure scenarios and measure MTTD and MTTR for each
5. Analyse the impact of observability on recovery speed

## 1.4 Scope

- Platform: AWS Free Tier (EC2 t2.micro)
- Stack: Nginx, Docker, GitHub Actions, CloudWatch, SNS
- Failure types: container crash, CPU spike, memory exhaustion,
  disk pressure, health check failure
- Duration: controlled lab environment, not production traffic

## 1.5 Organisation of Report

Chapter 2 reviews existing literature on observability and MTTR.
Chapter 3 covers system design and architecture.
Chapter 4 details implementation.
Chapter 5 presents failure simulation results and MTTD/MTTR analysis.
Chapter 6 concludes with findings and future scope.