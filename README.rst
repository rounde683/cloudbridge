# Setup - Coiled Q/A

At Coiled we're trying to launch dask clusters on various cloud providers (aws,
gcp, azure, etc).  Initially our clusters were all hosted on  AWS ECS code but
we needed to support additional providers (gcp, azure, etc) so we looked for a
python library that could simplify managing resources on multiple providers.

We ended up using [cloudbridge](https://github.com/CloudVE/cloudbridge), which
seems to be a nice solution since it's a single python API that wraps multiple
provider APIs.  But it turns out that the functionality in cloudbridge does not
exactly match what we need so we had to go in and tweak it and that is what
this challenge is about.

##  Warm Up

Start by navigating to the cloudbridge
[repo](https://github.com/CloudVE/cloudbridge) and docs and go to [Launch an
instance
example](http://cloudbridge.cloudve.org/en/latest/getting_started.html#launch-an-instance)
in Getting Started.  There you will find the following snippet:

```
img = provider.compute.images.get(image_id)
vm_type = sorted([t for t in provider.compute.vm_types
                  if t.vcpus >= 2 and t.ram >= 4],
                  key=lambda x: x.vcpus*x.ram)[0]
inst = provider.compute.instances.create(
    image=img, vm_type=vm_type, label='cb-instance',
    subnet=sn, key_pair=kp, vm_firewalls=[fw])
# Wait until ready
inst.wait_till_ready()  # This is a blocking call
# Show instance state
inst.state
# 'running'

```

**Question 1:** please explain what the `vm_type` line is doing, in words.  Be
very specific and indicate what specific condition `vm_type` will satisfy.

**Answer 1:**
Vm_type is listing the instance types of a provider where the CPUS are 2 or greater and the RAM is 4GB or larger. 
It's sorted so it's selecting the first available vm type instance type name/id for that provider that matches the CPU and RAM condition.

### Bonus Question (depending on your familiarity with devops)

If you are familiar with AWS (or other providers) look down at the [Assign apublic IP address section]
(http://cloudbridge.cloudve.org/en/latest/getting_started.html#assign-a-public-ip-address)

```
if not inst.public_ips:
    fip = gateway.floating_ips.create()
    inst.add_floating_ip(fip)
    inst.refresh()
inst.public_ips
# [u'54.166.125.219']

```
and answer the following questions:

**Question 2:** What is this code snippet doing?  What is a "floating" IP in terms of a specific cloud provider?  Please answer using terms from your favorite cloud provider (gcp, aws, etc).

**Answer 2**
The code snippet is adding a "Floating IP" to the instance. Tracing the code to the call itself, it is attaching an elastic IP to the instance in GCP, Azure, and AWS.
cloudbridge\prividers\aws
   def add_floating_ip(self, floating_ip):
cloudbridge\prividers\azure
   def add_floating_ip(self, floating_ip):
cloudbridge\prividers\gcp
   def add_floating_ip(self, floating_ip):


**Question 3:** Is there any reason why using a 'floating' IP might not make sense for Coiled?  Recall that Coiled launches many **transient** dask clusters for clients and those cluster each need a public IP.

**Answer 3**
Using only elastic IPs wouldn't scale to meet Coiled's needs. 
From what I was able to briefly google accross GCP, Azure, and AWS. Each of these providers has a limitation on the number of Elastic IPs per VPC or Region. In AWS elastic IPs are limited to 5 per region. I'd imagine this model wouldn't probe to be profitable as AWS has 25 geographic regions and 80 AV zones. You'd have a hard limit of 5*25*80 = 10,000 'pods' at any given time.



## Coding Tasks

For reasons you may have figured out above we do not want to add "floating" IPs
to instances after launching them.  Rather we want to launch instances with
dynamically generated public IPs.  Specifically we want to modify the instance
creation call above to look like:

```
inst = provider.compute.instances.create(
    image=img, vm_type=vm_type, label='cb-instance',
    subnet=sn, key_pair=kp, vm_firewalls=[fw],
public_ip=True)

```

Unfortunately cloudbridge itself does not support a `public_ip` flag like this:
VMs launched using the cloudbridge API always have private IP addresses and can
only be made publicly accessible by attaching floating IPs as above (which we
do not want to do).  So to enable the behavior we want above we need to modify
the cloudbridge library.

**Task 1:** Modify the cloudbridge library so that the code-snippet above works
and launches a VM with a dynamic public IP.  Implement this for your favourite
provider in cloudbridge (aws, gcp, azure).

**Task 2:** Optionally implement the same logic across all 3 providers.














CloudBridge provides a consistent layer of abstraction over different
Infrastructure-as-a-Service cloud providers, reducing or eliminating the need
to write conditional code for each cloud.

Documentation
~~~~~~~~~~~~~
Detailed documentation can be found at http://cloudbridge.cloudve.org.


Build Status Tests
~~~~~~~~~~~~~~~~~~
.. image:: https://github.com/CloudVE/cloudbridge/actions/workflows/integration.yaml/badge.svg
   :target: https://github.com/CloudVE/cloudbridge/actions/
   :alt: Integration Tests

.. image:: https://codecov.io/gh/CloudVE/cloudbridge/branch/master/graph/badge.svg
   :target: https://codecov.io/gh/CloudVE/cloudbridge
   :alt: Code Coverage

.. image:: https://img.shields.io/pypi/v/cloudbridge.svg
   :target: https://pypi.python.org/pypi/cloudbridge/
   :alt: latest version available on PyPI

.. image:: https://readthedocs.org/projects/cloudbridge/badge/?version=latest
   :target: http://cloudbridge.readthedocs.org/en/latest/?badge=latest
   :alt: Documentation Status

.. |aws-py36| image:: https://travis-matrix-badges.herokuapp.com/repos/CloudVE/cloudbridge/branches/master/1?use_travis_com=yes
              :target: https://travis-ci.com/CloudVE/cloudbridge

.. |azure-py36| image:: https://travis-matrix-badges.herokuapp.com/repos/CloudVE/cloudbridge/branches/master/2?use_travis_com=yes
                :target: https://travis-ci.com/CloudVE/cloudbridge

.. |gcp-py36| image:: https://travis-matrix-badges.herokuapp.com/repos/CloudVE/cloudbridge/branches/master/3?use_travis_com=yes
              :target: https://travis-ci.com/CloudVE/cloudbridge

.. |mock-py36| image:: https://travis-matrix-badges.herokuapp.com/repos/CloudVE/cloudbridge/branches/master/4?use_travis_com=yes
              :target: https://travis-ci.com/CloudVE/cloudbridge

.. |os-py36| image:: https://travis-matrix-badges.herokuapp.com/repos/CloudVE/cloudbridge/branches/master/5?use_travis_com=yes
             :target: https://travis-ci.com/CloudVE/cloudbridge

+---------------------------+----------------+
| **Provider/Environment**  | **Python 3.6** |
+---------------------------+----------------+
| **Amazon Web Services**   | |aws-py36|     |
+---------------------------+----------------+
| **Google Cloud Platform** | |gcp-py36|     |
+---------------------------+----------------+
| **Microsoft Azure**       | |azure-py36|   |
+---------------------------+----------------+
| **OpenStack**             | |os-py36|      |
+---------------------------+----------------+
| **Mock Provider**         | |mock-py36|    |
+---------------------------+----------------+

Installation
~~~~~~~~~~~~
Install the latest release from PyPi:

.. code-block:: shell

  pip install cloudbridge

For other installation options, see the `installation page`_ in
the documentation.


Usage example
~~~~~~~~~~~~~

To `get started`_ with CloudBridge, export your cloud access credentials
(e.g., AWS_ACCESS_KEY and AWS_SECRET_KEY for your AWS credentials) and start
exploring the API:

.. code-block:: python

  from cloudbridge.factory import CloudProviderFactory, ProviderList

  provider = CloudProviderFactory().create_provider(ProviderList.AWS, {})
  print(provider.security.key_pairs.list())

The exact same command (as well as any other CloudBridge method) will run with
any of the supported providers: ``ProviderList.[AWS | AZURE | GCP | OPENSTACK]``!


Citation
~~~~~~~~

N. Goonasekera, A. Lonie, J. Taylor, and E. Afgan,
"CloudBridge: a Simple Cross-Cloud Python Library,"
presented at the Proceedings of the XSEDE16 Conference on Diversity, Big Data, and Science at Scale, Miami, USA, 2016.
DOI: http://dx.doi.org/10.1145/2949550.2949648


Quick Reference
~~~~~~~~~~~~~~~
The following object graph shows how to access various provider services, and the resource
that they return.

.. image:: http://cloudbridge.readthedocs.org/en/latest/_images/object_relationships_detailed.svg
   :target: http://cloudbridge.readthedocs.org/en/latest/?badge=latest#quick-reference
   :alt: CloudBridge Quick Reference


Design Goals
~~~~~~~~~~~~

1. Create a cloud abstraction layer which minimises or eliminates the need for
   cloud specific special casing (i.e., Not require clients to write
   ``if EC2 do x else if OPENSTACK do y``.)

2. Have a suite of conformance tests which are comprehensive enough that goal
   1 can be achieved. This would also mean that clients need not manually test
   against each provider to make sure their application is compatible.

3. Opt for a minimum set of features that a cloud provider will support,
   instead of  a lowest common denominator approach. This means that reasonably
   mature clouds like Amazon and OpenStack are used as the benchmark against
   which functionality & features are determined. Therefore, there is a
   definite expectation that the cloud infrastructure will support a compute
   service with support for images and snapshots and various machine sizes.
   The cloud infrastructure will very likely support block storage, although
   this is currently optional. It may optionally support object storage.

4. Make the CloudBridge layer as thin as possible without compromising goal 1.
   By wrapping the cloud provider's native SDK and doing the minimal work
   necessary to adapt the interface, we can achieve greater development speed
   and reliability since the native provider SDK is most likely to have both
   properties.


Contributing
~~~~~~~~~~~~
Community contributions for any part of the project are welcome. If you have
a completely new idea or would like to bounce your idea before moving forward
with the implementation, feel free to create an issue to start a discussion.

Contributions should come in the form of a pull request. We strive for 100% test
coverage so code will only be accepted if it comes with appropriate tests and it
does not break existing functionality. Further, the code needs to be well
documented and all methods have docstrings. We are largely adhering to the
`PEP8 style guide`_ with 80 character lines, 4-space indentation (spaces
instead of tabs), explicit, one-per-line imports among others. Please keep the
style consistent with the rest of the project.

Conceptually, the library is laid out such that there is a factory used to
create a reference to a cloud provider. Each provider offers a set of services
and resources. Services typically perform actions while resources offer
information (and can act on itself, when appropriate). The structure of each
object is defined via an abstract interface (see
``cloudbridge/providers/interfaces``) and any object should implement the
defined interface. If adding a completely new provider, take a look at the
`provider development page`_ in the documentation.


.. _`installation page`: http://cloudbridge.readthedocs.org/en/
   latest/topics/install.html
.. _`get started`: http://cloudbridge.readthedocs.org/en/latest/
    getting_started.html
.. _`PEP8 style guide`: https://www.python.org/dev/peps/pep-0008/
.. _`provider development page`: http://cloudbridge.readthedocs.org/
   en/latest/
    topics/provider_development.html
