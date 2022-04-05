# Authoring Supply Chains

The Out of the Box Supply Chain, Delivery Basic and Templates packages provide
a set of Kubernetes objects aiming at covering a reference path to production
prescribed by VMware, but recognizing that each organization will have their
own needs, any of those objects can be customized, from the individual
templates for each resource, to the whole supply chains.

Depending on how TAP has been installed, there will be different ways of
achieving the customization of the out of the box supply chains.

Below you'll find sections covering different setups and how to proceed.


## TAP Profiles

When installing TAP making use of profiles, a PackageInstall object is created
defining that a particular set of TAP components should be installed. 

```
# add the definition of all the PackageMetadata and Package objects for this
# version of TAP
tanzu package repository add <tap>
```

```
# create the PackageInstall object that installs a certain version of TAP,
# which then translates to a series of PackageInstall objects (one for each
# package) being submitted
#
tanzu package install tap --values-file tap-values.yaml
```


Being the installation based on Kubernetes primitives, PackageInstall will
relentlessly try to achieve the state of having all the packages installed.
This is great in overall, but it presents some challanges for modifying the
contents of some of the objects that the installation submits to the cluster:
any live modifications to them will result in the original definition being
persisted instead of the changes.



* TAP-specific details (pause package installation .. renaming ...)

    --> if you used profiles:
    --> if you didn't use profiles but did `tanzu package install`


### Profile-based installation

1. pause TAP package install


### Component-based installation

1. pause the PackageInstall for `ootb-templates`, if wanting to change
   templates

1. pause the PackageInstall for `ootb-supply-chain-<>` for changing
   supplychains



## Cartographer


https://cartographer.sh/docs/development/tutorials/first-supply-chain/
