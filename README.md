
[//]: # ( Copyright 2017 Turbine Labs, Inc.                                   )
[//]: # ( you may not use this file except in compliance with the License.    )
[//]: # ( You may obtain a copy of the License at                             )
[//]: # (                                                                     )
[//]: # (     http://www.apache.org/licenses/LICENSE-2.0                      )
[//]: # (                                                                     )
[//]: # ( Unless required by applicable law or agreed to in writing, software )
[//]: # ( distributed under the License is distributed on an "AS IS" BASIS,   )
[//]: # ( WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or     )
[//]: # ( implied. See the License for the specific language governing        )
[//]: # ( permissions and limitations under the License.                      )

# CloudFormation + NLB + Envoy + Houston

This repository walks through installing and configuring Envoy behind an
[AWS Network Load Balancer](http://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html).
At the end of the exercise you'll have
* an NLB sending all traffic to a pool of Envoy proxies in an autoscaling group
* Envoy routing traffic to two different backing services, also in autoscaling
  groups
* Houston providing xDS implementations that keep Envoy in sync with autoscaling
  group changes, along with a straightforward UI for configuring Envoy routes
* Houston displaying customer-centric metrics and a log of all changes made to
  traffic configuration

Creating the CloudFormation stacks will deploy two services. The client
application presents a simple UI that shows blinking boxes. The server
application returns a hex color that the boxes should be colored. CloudFormation
will also deploy tbnproxy, running Envoy, configured as a target group for a
network load balancer. It will also deploy tbncollect to track the configuration
of your environment. With these pieces in place you will use Houston to
configure routes to map appropriate traffic to the client and server services.

## Prerequisites

Note: you can create a single stack with both the application and Envoy by using
the cloud-formation-combined.yaml file. However We recommend separating the
stacks as it more accurately reflects a production deployment where the
application and proxy pools are configured and released independently. If you
prefer Terraform to CloudFormation there are equivalent Terraform files included
in the repository.

Before getting started you'll need the following:

### An AWS Account

You can [sign up for an account](https://aws.amazon.com/) if you don't have
one. You'll need permissions to create VPCs, NLBs, EC2 instances, and
autoscaling groups. You'll also need
[an access key](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html).

### A VPC

To launch the stack you'll need a VPC configured with at least two subnets in
different availability zones

### A Turbine Labs Account

If you don't have an account, [contact us](https://www.turbinelabs.io/contact)
for a free trial.

### A Working Golang Installation

[Find a package](https://golang.org/doc/install), and ensure it's properly
configured.

### tbnctl

`tbnctl` is a CLI written in go that lets you work with the Turbine Labs data
model. You can install it with:

```console
$ go get -u github.com/turbinelabs/tbnctl
$ go install github.com/turbinelabs/tbnctl
```

and then authenticate with the Turbine Labs API

```console
$ tbnctl login
```

Next, create an access token

```console
$ tbnctl access-tokens add "nlb demo"
```

The result should be something like

```json
{
  "access_token_key": "<redacted>",
  "description": "demo key",
  "signed_token": "<redacted>",
  "user_key": "<redacted>",
  "created_at": "2017-08-25T22:11:30.907200482Z",
  "checksum": "d60ed8a6-1a40-49a5-5bb1-5bad322d9723"
}
```

Save the `signed_token` field to use later.

# Getting Started

## Creating a Houston zone and proxy

First create a zone for this demo by running

```console
$tbnctl init-zone nlb-demo
```

Next, create a proxy named 'nlb-demo', following the
[instructions here](https://docs.turbinelabs.io/guides/ec2.html#adding-a-domain-and-proxy).
Note that this example creates a proxy called 'testbed-proxy'. Yours should be
named 'nlb-demo' instead.

## Setting up the Envoy stack

Go to the [AWS console](https://console.aws.amazon.com), select the region in
which you wish to launch your stack, and click "CloudFormation" under the
Management Tools section.

Click "Create Stack" in the top left section of the screen. In the "Select
Template" screen choose "Upload a template to Amazon S3", click "choose file",
and select the cloud-formation-envoy.yaml file in this repository. Then click
Next

In the Specify Details screen fill in appropriate variables. You can name your
stack anything you like. You will need to select two subnets, each running in
different Availability Zones to provide redundancy for your proxy pool. The
TbnZoneName must be a valid Houston zone, and the TbnProxyName must be the name
of a proxy configured in that zone.

Click through the rest of the screens, and launch your stack. Note that
provisioning a load balancer in AWS can take several minutes. When the stack is
created, move on to domain creation.

## Deploying the application

Go to the [AWS console](https://console.aws.amazon.com) again, select the region
in  which you wish to launch your stack, and click "CloudFormation" under the
Management Tools section.

Click "Create Stack" in the top left section of the screen. In the "Select
Template" screen choose "Upload a template to Amazon S3", click "choose file",
and select the cloud-formation-client.yaml file in this repository. Then click
Next

In the Specify Details screen fill in appropriate variables. You can name your
stack anything you like, but the security group you choose must include the
"InstanceSecurityGroup" created as part of the Envoy stack

Create another stack for the server application by repeat these steps using the
cloud-formation-server.yaml file.

## Adding a Houston domain

The CloudFormation stack should have the hostname of the created LoadBalancer in
its "outputs". Create a corresponding Houston domain following the
[instructions here](https://docs.turbinelabs.io/guides/ec2.html#adding-a-domain-and-proxy),
replacing the domain name with yours. Make sure you select the nlb-proxy in the
'Proxies' box during domain creation. This tells Houston to configure Envoys
named nlb-proxy to serve the configured domain.

## Configuring Routes

Now configure routes to send traffic to the client and server applications
terraform deployed. First we'll configure a route to send all traffic to the
client server.

* Make sure you have the 'nlb-demo' zone selected in the top left portion of the screen.
* Click the "Settings" menu in the top right portion of the screen, and then select "Edit Routes".
* Click the "More" menu, then select "Add Route".
* Select your domain in the domain drop down
* Enter "/" in the path field
* Click the release group dropdown and select "Create New Release Group..."
* Select "client" from the service drop down
* Enter "client" in the release group name field
* Click the "Create Route + Release Group" button

And next we'll create one to send anything for /api to the server application.

* Make sure you have the 'nlb-demo' zone selected in the top left portion of the screen.
* Click the "Settings" menu in the top right portion of the screen, and then select "Edit Routes".
* Click the "More" menu, then select "Add Route".
* Select your domain in the domain drop down
* Enter "/api" in the path field
* Click the release group dropdown and select "Create New Release Group..."
* Select "server" from the service drop down
* Enter "server" in the release group name field
* Click the "Create Route + Release Group" button

## Success

Now you should be able to visit your domain, e.g.

`http://cfenv-loadb-13m1i0qii67te-37085f4fe3c82383.elb.us-west-1.amazonaws.com`

and see your demo working.

## Cleanup

You can remove all created resources by running choosing to delete the stack in
the CloudFormation UI
