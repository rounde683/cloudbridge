Setup - Coiled Q/A
~~~~~~~~~~~~~

At Coiled we're trying to launch dask clusters on various cloud providers (aws,
gcp, azure, etc).  Initially our clusters were all hosted on  AWS ECS code but
we needed to support additional providers (gcp, azure, etc) so we looked for a
python library that could simplify managing resources on multiple providers.

We ended up using [cloudbridge](https://github.com/CloudVE/cloudbridge), which
seems to be a nice solution since it's a single python API that wraps multiple
provider APIs.  But it turns out that the functionality in cloudbridge does not
exactly match what we need so we had to go in and tweak it and that is what
this challenge is about.

Warm Up
~~~~~~~~~~~~~

Start by navigating to the cloudbridge
[repo](https://github.com/CloudVE/cloudbridge) and docs and go to [Launch an
instance
example](http://cloudbridge.cloudve.org/en/latest/getting_started.html#launch-an-instance)
in Getting Started.  There you will find the following snippet:

.. code-block:: python

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



Question 1:
~~~~~~~~~~~~~
Please explain what the `vm_type` line is doing, in words.  Be
very specific and indicate what specific condition `vm_type` will satisfy.


Answer 1:
~~~~~~~~~~~~~
Vm_type is listing the instance types of a provider where the CPUS are 2 or greater and the RAM is 4GB or larger. 
It's sorted so it's selecting the first available vm type instance type name/id for that provider that matches the CPU and RAM condition.

Bonus Question (depending on your familiarity with devops)
~~~~~~~~~~~~~

If you are familiar with AWS (or other providers) look down at the [Assign apublic IP address section]
(http://cloudbridge.cloudve.org/en/latest/getting_started.html#assign-a-public-ip-address)

.. code-block:: python

  if not inst.public_ips:
    fip = gateway.floating_ips.create()
    inst.add_floating_ip(fip)
    inst.refresh()
  inst.public_ips
  # [u'54.166.125.219']


and answer the following questions:

Question 2:
~~~~~~~~~~~~~
What is this code snippet doing?  What is a "floating" IP in terms of a specific cloud provider?  Please answer using terms from your favorite cloud provider (gcp, aws, etc).

Answer 2
~~~~~~~~~~~~~
The code snippet is adding a "Floating IP" to the instance. Tracing the code to the call itself, it is attaching an elastic IP to the instance in GCP, Azure, and AWS.
.. code-block:: python
  cloudbridge\prividers\aws
    def add_floating_ip(self, floating_ip):
  cloudbridge\prividers\azure
    def add_floating_ip(self, floating_ip):
  cloudbridge\prividers\gcp
    def add_floating_ip(self, floating_ip):


Question 3:
~~~~~~~~~~~~~
Is there any reason why using a 'floating' IP might not make sense for Coiled?  Recall that Coiled launches many **transient** dask clusters for clients and those cluster each need a public IP.

Answer 3
~~~~~~~~~~~~~
Using only elastic IPs wouldn't scale to meet Coiled's needs. 
From what I was able to briefly google accross GCP, Azure, and AWS. Each of these providers has a limitation on the number of Elastic IPs per VPC or Region. In AWS elastic IPs are limited to 5 per region. I'd imagine this model wouldn't probe to be profitable as AWS has 25 geographic regions and 80 AV zones. You'd have a hard limit of 5*25*80 = 10,000 public IPs at any given time.

A 10,000 limitation on customers is not good enough to be profitable, I'd imagine.



Coding Tasks
~~~~~~~~~~~~~

For reasons you may have figured out above we do not want to add "floating" IPs
to instances after launching them.  Rather we want to launch instances with
dynamically generated public IPs.  Specifically we want to modify the instance
creation call above to look like:

.. code-block:: python

  inst = provider.compute.instances.create(
     image=img, vm_type=vm_type, label='cb-instance',
     subnet=sn, key_pair=kp, vm_firewalls=[fw],
  public_ip=True)


Unfortunately cloudbridge itself does not support a `public_ip` flag like this:
VMs launched using the cloudbridge API always have private IP addresses and can
only be made publicly accessible by attaching floating IPs as above (which we
do not want to do).  So to enable the behavior we want above we need to modify
the cloudbridge library.

Task 1:
~~~~~~~~~~~~~
Modify the cloudbridge library so that the code-snippet above works
and launches a VM with a dynamic public IP.  Implement this for your favourite
provider in cloudbridge (aws, gcp, azure).

Task 2:
~~~~~~~~~~~~~
Optionally implement the same logic across all 3 providers.
