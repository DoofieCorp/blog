+++
categories = []
date = "2016-06-28T17:10:58+01:00"
description = "Introduction blog post"
keywords = ["terraform", "hashicorp", "vagrant", "packer", "vault", "devops", "security", "aws", "azure", "consul"]
title = "Hello World"

+++

```ruby
resource "null_resource" "hello_world" {
    provisioner "local-exec" {
        command = "echo 'Hello World'"
    }
}
```

Interested in automation and the HashiCorp suite of tools? If so you'll love this blog.
Through different posts we will explore lots different automation tasks utilising both
public and private cloud with the HashiCorp toolset.

Thanks,
Ian.
