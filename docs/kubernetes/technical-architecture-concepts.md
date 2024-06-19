# Kubernetes Technical Architecture Concepts

This page explains the basic concepts of the Kubernetes technical architecture which I find very important to better understand Kubernetes as a whole. 

## Goal of this page

Kubernetes has evolved from just being a container scheduling and management system. It can be used as a generic "platform API" - a standardized API for an entire platform consisting of not only containers, but also virtual machines, databases and services. The reason for this success is due to the well architected architecture in my opinion.

This is the reason why I think it is very valuable to understand the basic technical concepts as it will help you better understand literally *anything*. 

I try to go into technical details without going into technical details :wink:.

We will cover:

- what actually happens when I create a Kubernetes deployment?
- Kubernetes reconciliation and controllers
- Kubernetes Admission Webhooks - Mutating and Validating
- Custom Resource Definitions (CRDs)

## 
