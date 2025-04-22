+++
date = '2025-04-17T07:51:52-04:00'
draft = false
title = 'AWS EKS Auto Mode and EBS CSI Driver Topology Issues'
+++

I just found something really interesting out regarding AWS EKS Auto Mode and the EBS CSI Driver.  I deployed EKS Auto Mode and used the Add On made available to provision the EBS CSI Driver with EKS Pod Identity.  

I spent way too many hours rebuilding clusters, tweaking things, playing with IAM Roles, removing and re-adding Add Ons.  I was going crazy.

Not gonna lie I started by using ChatGPT.  In an effort to be extremely thorough I started putting everything through the wringer, including the recommended Policy and Trust relationships for the Add-Ons.  For the EBS CSI Driver role ChatGPT had an aha moment looking at the Trust relationship.  What starts as below is wrong:

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "pod.eks.amazonaws.com"
                },
                "Action": [
                    "sts:AssumeRole",
                    "sts:TagSession"
                ]
            }
        ]
    }

In fact the Service should be `"Service": "eks.amazonaws.com"`

This got me closer, but I wasn't done.Frustrated I could not get a PVC to provision a PV successfully and had all manner of errors realted to topology problems.  Relevant message from the kubernetes API was:

    failed to provision volume with StorageClass "gp2": error generating accessibility requirements: no topology key found on CSINode i-02515766f5107ad3b



Then I found an [AWS example](https://docs.aws.amazon.com/eks/latest/userguide/sample-storage-workload.html) for this and they used a different StorageClass that looks like:


    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: auto-ebs-sc
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: ebs.csi.eks.amazonaws.com
    volumeBindingMode: WaitForFirstConsumer
    parameters:
      type: gp3
      encrypted: "true"


  
So I deployed this, made it default storageclass, and voila my PVCs started spinning up PVs.  Boom.  Seems a glaring hole in AWS's tech right now, though, and I'm surprised I haven't found more information online regarding this issue.  I'm documenting here as much for my own memory as anything else.

