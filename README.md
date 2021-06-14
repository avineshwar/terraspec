# Terraspec

Terraspec is a unit test framework for terraform configurations. It lets you quickly check what your terraform configurations will create for a specific set of input variables.

Terraspec is an additional tool to your terraform toolbox. It doesn't provision any resource in your cloud provider nor does it let you test your cloud resources. Terraspec only tests the code you write in your config.

For those aware of Chef development, `terraspec` aims to be an equivalent to `chefspec` rather than `test-kitchen`.

## Usage

### Write your tests

In your terraform configuration directory, create a `spec` folder and a subfolder for every test scenario.

At least, your test suite subfolder must contain a `.tfspec` file containing all the assertions on your code. 
To test a different scenario than the  default input variables, you can provide a `.tfvars` file as well.

**Examples are available in the `examples` directory of this repository.**

Writing an assertion is as easy as writing your initial terraform configuration. If you want to check the behaivor of this terraform code :

```hcl
resource "aws_instance" "my-server" {
    ami = var.ami
    ...
}
```

You can write the following code in your `tfspec` file : 
```hcl
assert "aws_instance" "my-server" {
    ami = "the-ami-value-expected"
}
```

To test the value of an output, you can write :
```hcl
assert "output" "output-name" {
    value = "expected-output-value"
}
```

You can also check a resource won't be created with this syntax : 
```hcl
reject "aws_instance" "another-server" {}
```

Or you can check a resource will be created **without** specific configuration, eg : 
 ```hcl
assert "aws_instance" "my-server" { // resource my-server must exist

  reject {
    // my-server must not have any ebs_block_device set
    ebs_block_device{} 

    // my-server must not have any credit_specification with cpu_credits set to "unlimited"
    credit_specification{
      cpu_credits = "unlimited" 
    }
  }
}
```

### Expect resource attributes

Writing assertions not only lets your specify test about the expected arguments on resource creation, but it can also let you mock the return attributes. To do so, add a `return` block in the `assert` one and set the attribute values you want to be returned.

Example: 

```hcl
assert "aws_s3_bucket" "backup_bucket" {
    bucket = "testbucket"
    return {
      id = "arn:aws:s3::testbucket"
      arn = "arn:aws:s3::testbucket"
    }
}
```

`assert` or `expect` : it's up to you. Terraspec support both keywords to specify the assertions. You can use `assert` for assertion only and `expect` when it also mocks returned attributes or you can stick with one keyword for all use cases.


### Mock data resource

If your configuration contains `data` resource, you can mock their value by writing a `mock` resource in your spec file. A `mock` resource must have the exact same configuration block as the `data` resource. The data you want to return must be set in a `return` block.

Example :

Terraform code :
```hcl
data "aws_vpcs" "selected" {
  tags = {
    service = "secure"
  }
}
```

The spec file mocking this data source should be :
```hcl
mock "aws_vpcs" "selected" {
  // Config block must be duplicated
  tags = {
    service = "secure"
  }
  return {
    // All attributes of the returned data are set in the return block
    ids = ["mocked_vpc_id"]
  }
}
```

### Mock data resource with different provider
In some situations you may declare multiple providers and call the same datasource with each of them. So your terraform code look like :

```hcl
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias = "eu-west-2"
  region = "eu-west-2"
}

data "aws_region" "west1" {}
data "aws_region" "west2" {
    provider = aws.eu-west-2
}
```

In such situation, you can specify different mocks, using the `provider` attribute of mock resources. For example, your terraspec code would be :

```hcl
mock "aws_region" "west1"{
    return {
        name = "ireland"
    }
}
mock "aws_region" "west2"{
    provider = "aws.eu-west-2"
    return {
        name = "london"
    }
}
```

Note that the provider value is a string containing the provider name (aws) and its alias (eu-west-2) separated by a dot.

### Terraform Workspace

If you want to use the terraform workspace feature in terraspec you need to first configure which workspace value to use. You can do this in a spec global element `terraspec`:

__default.tfspec__
```hcl
# define which workspace you want to test
terraspec {
    workspace = "development"
}

# use the workspace value in your test assertions or anywhere else
assert "aws_vpc" "test_vpc" {
    tags = {
        "env": terraspec.workspace
    }
}
```

The stated workspace value will also be injected into the terraform configuration that is tested.

See also [examples/workspace](examples/workspace).

### Run 

To call `terraspec`, you must have run `terraform init` first to have all the plugins and modules downloaded. 
The first time you write `terraspec` should be like : 
```
$ terraform init -backend=false
$ terraspec 
```
As terraspec will never try to read your current state, you don't even need to init the remote backend.

If you want to run a single test scenario, you can specify it with the `--spec` flag : 
```
$ terraspec --spec spec/my-scenario
```

The command line flag `--diplay-plan` can help to write your tests. As name suggests, with this flag `terraspec` will print you the output of `terraform plan`. 


## Use cases

The examples given so far are really easy and can seem useless. However there a re situations where writing this kind of tests is really helpful :

- When your configuration do tricky operations with terraform provided function like splitting a dns name to find the DNS zone to update, or doing computation on an IP address or subnet mask, ...
- When your configuration may or may not create additionnal resources, depending on advanced conditions
- When you want to check all your tags have been correctly merged and applied to your resources
- ...

Also, as a module author, it can be interesting to ensure the new version of your module won't have side effects so users can update peacefully.

## How it works

Terraspec embeds terraform code, so even if it doesn't make call to the `terraform` command, it relies on `terraform` to compute the plan. Nevertheless, `terraspec` wraps all calls to the underlying plugin so that the terraform state is never read, nor the `data` resource.
This makes `terraspec` able to validate any configuration, whichever cloud provider you use, without any credentials to that cloud provider.

## Limitations

Terraspec is still at its early stages and doesn't cover all cases yet. Here are the known limitations identified so far.



### Terraform version constraints

When you run `terraspec`, the version constraint set in your plan will be checked with the version of `terraform` embedded in `terraspec`. This means that if your `terraform` config defines a strict constraint about which `terraform` version it supports, the version of `terraform` embedded in `terraspec` may not comply with it.

To help with you :
- Upgrading `terraform` whenever you want, regardless there's a `terraspec` version matching
- Upgrading `terraspec` whenever you want, regardless the version constraint set in your config

`terraspec` provides a `--claim-version` flag. This flag will tell `terraspec` to substitute the `terraform` version defined in the code so it can comply with the version constraint.

Use with care : this flag won't change the version of `terraform` effectively used to parse your code when testing it with `terraspec`. Using a highly different version of `terraform` than the one embedded in `terraspec` may lead to wrong validation.

To know which vesion of `terraform` is embedded in `terraspec`, run `terraspec --version`.

### AWS Provider v3
Starting with version 3 of aws provider, it is required to run the provider configuration step. This means that when running `terraspec` you need to be able to authenticate to aws as you would when running a `terraform plan`, for instance.


By default, `terraspec` won't run the provider configuration phase. So when your terraform config uses aws provider in version 3, your need to run terraspec with `--configure-provider` flag.

If you use aws provider v3 and don't use `--configure-provider`, `terraspec` will error with this message (meaning the aws plugin crashed):

```
rpc error: code = Unavailable desc = transport is closing :
```

## Installation

For gophers, running `go get github.com/nhurel/terraspec` should do the trick.

Otherwise, download a released binary from the releases page, put it in your PATH and make sure it's executable

## License

Mozilla Public License 2.0




