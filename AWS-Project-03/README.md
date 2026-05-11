# AWS Project-03: EC2 with Application Load Balancer vs S3 Static Website Hosting — When to Use Each on AWS

Cloud computing has changed the way we build and host applications. Some of the ways you can achieve this on AWS is by either running your application on a compute-based service like EC2 or serve files directly form storage-based services like S3.

In this project, I’ll walk you through deploying a simple HTML page using the two approaches:

1.  Compute-based hosting with EC2 instances behind an Application Load Balancer (ALB). Here, we’ll be making use of Ansible automation to install and configure Nginx, and deploy our website across multiple instances (servers).
2.  Storage-based hosting with S3 static webseite hosting

By the end, you’ll not only have deployed your first AWS project but also understand the key differences between compute and storage hosting and also why one might be better than the other depending on your use case.

Prerequisites
-------------

To follow along, you only need:

*   An AWS account
*   Ansible installed on your local machine to automate
*   Any code editor of your choice
*   SSH key pair for connecting to your EC2 instances (you’ll download this when creating the instances)

We’ll use Ansible for the compute-based deployment (EC2 + ALB), so you’ll see how automation simplifies installing Nginx and deploying the same page across multiple servers.

Step 1: Launch 2 EC2 instances
------------------------------

We’ll start by creating two Ubuntu EC2 instances on AWS.

For the purpose of this project, I’ll be making use of the following setup:

*   **AMI**: Ubuntu 24.04
*   **Instance type**: t2.micro
*   **Security group**: allow inbound SSH (port 22) and HTTP (port 80)

From your AWS console, navigate to the EC2 dashboard.

This can be done by clicking on the services menu, selecting **Compute** and then **EC2** option from the dropdown menu.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*EpBmDHvdBj6F_ywKMTrACA.png)

Once you’re in the dashboard, select the ‘**Launch Instance**’ button. This opens up a wizard to setup your **EC2 instance.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*_Zwro7ZXe9Vdy37uP-i24g.png)

Add a unique name for your instance and select the the necessary AMI (Ubuntu) and instance type (t2.micro).

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*c6EH4v6TJIqsb3Ao6s0BMw.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Url3-yDCsbFEeMQMX_1_mQ.png)

Next we create a new SSH key pair. If you already have one, you can select that and skip this part.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*JxX8anKx_yTXX3kuiCzsCg.png)

Add a name and then click on the ‘**Create key pair**’ button. This action would automatically select the new key pair and downloaded to your local machine (check your Downloads folder).

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*-eoT8O8wZd4O1wTZoPKgIQ.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*1HzdLH776kfyGArDLtwViQ.png)

Next up is the network settings. Click on the ‘**Edit**’ button, so we can update some extra attributes.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HPm3i3TviFP7nx5HKcvrlA.png)

Leave the other attributes as they are and select the ‘**Availability Zone**’ dropdown. Select any option from the dropdown and note it down. We would need this later when setting up our load-balancer.

Here, I’ll be choosing ‘us-east-1a’.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*sO5iFaPvtTQL_9rnCuj68w.png)

Still in the ‘**Network settings**’, for the **Firewall (security groups)** section, select the ‘**Create security group**’ option. This allows us to setup inbound rules for SSH and HTTP access.

Set a suitable **‘Security group name**’ and ‘**Description**’.

For the ‘**Inbound Security Group Rules’**, add ‘ssh’ and ‘HTTP’ from anywhere as seen in the description below.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TUpYiFECSE-N91RBPMvxUw.png)

Finally, at the ‘**Summary**’, click on ‘**Launch instance**’ to start the instance creation process.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*eKlPHfQdO_4ITbUZkCV8fw.png)

Once the process is finished, click on this link to open up your new EC2 instance.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*vZWxVe6pef27iciD-9w-8w.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*3LM-d_BIIjCm4TrX6MAH3g.png)

That wasn’t quite hard, was it?

Now repeat the same step for the second instance with only these two tweaks:

*   In the ‘**Key pair**’ section, no need to create another one. You can make use of the one created while setting up the first instance. It can also be used for any subsequent EC2 instances you create.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*1ki1myXnALF2v2NoNsS1Mg.png)

*   For the ‘**Network settings’**, click the ‘Edit’ button again in order to select any ‘**Availability Zone’** of your choice. Here, I’ll be choosing ‘us-east-1b’.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*9SiC15TRBsyXahSja6y15Q.png)

*   For the ‘**Firewall (security groups)**’, rather than create a new one as well, we can pick the one we creating previously by selecting the ‘**Select existing security group**’ option and then pick the security group we created previously.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*u0jqtVs0UdW0s50xCwnwjQ.png)

Finally, just like the first EC2 instance, in the ‘**Summary**’ section, click on the ‘**Launch instance**’ button to create the second instance.

Take note of the Public IPv4 addresses of both instances, we would be needing that in the next step.

Step 2: Deploy the HTML page on EC2
-----------------------------------

At this point, you have the two EC2 instances running but visiting it via its public IPv4 address (like this http://<ipv4-address>) will throw an error. This is because we haven’t setup any http webserver inside it to serve our files. For this project, we’ll be making use of [Nginx](https://nginx.org/).

To install Nginx, we can either do it manually by ssh-ing into each instance, manually installing and configuring Nginx or automate it using ansible.

Before proceeding with any of the options below, in order to avoid an error as seen in the screenshot below preventing you from ssh-ing into the server.

![captionless image](https://miro.medium.com/v2/resize:fit:1246/format:webp/1*5WwYF9TjaRW_pqPK1UUsAA.png)

Run the following command on your private ssh key:

```
chmod 400 /path/to/key-pair.pem
```

This is because OpenSSH refuses to use a private key if the file is too open (e.g., if others can read or write it).

### **Option A: Manual Deployment**

1.  SSH into each EC2 instance

```
ssh -i /path/to/key-pair.pem ubuntu@public_ipv4_address
```

2. Install Nginx manually

```
sudo apt update && sudo apt install -y nginx
```

3. Copy your HTML file int the Nginx web root

```
echo "<h1>Hello from this server</h1>" | sudo tee /var/www/html/index.html
```

4. Start and enable Nginx

```
sudo systemctl enable nginx
sudo systemctl start nginx
```

If done correctly, you should see an output like this on your browser when you visit ‘**http://<public_ipv4_address>**’.

Note that I used ‘**http**’ and not ‘**https**’.

![captionless image](https://miro.medium.com/v2/resize:fit:1152/format:webp/1*0TnGS4BoWWI8FVFf5oFojw.png)

### Option B: Automated Deployment with Ansible

If you want to avoid repeating the same steps across multiple servers, then this option is for you.

Ensure [**Ansible**](https://docs.ansible.com/) is already installed and running on your local machine.

In any directory of your choice, we’ll be creating the following files:

1.  **Inventory file (hosts.ini)**: public_ip_1 and public_ip_2 are the public ipv4 address of the first and second instances respectively.

```
[instances]
public_ip_1 ansible_user=ubuntu ansible_ssh_private_key_file=/path/to/key-pair.pem
public_ip_2 ansible_user=ubuntu ansible_ssh_private_key_file=/path/to/key-pair.pem
[instances:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

**⚠️ Security Note:** Disabling host key checking is fine for labs/tests/cloud instances you rebuild often. In production, it’s better to trust the fingerprint once (`ssh-keyscan` or manual accept by ssh-ing in manually first) instead of disabling it.

To confirm we can successfully ssh into the instances, run the command below:

```
ansible instances -i hosts.ini -m ping
```

We should get a success message like this.

![captionless image](https://miro.medium.com/v2/resize:fit:1212/format:webp/1*HFWSi35Lrwxo4oMhwBrT2g.png)

*   **Playbook file (setup.yml)**:

```
---
- name: Setup Nginx on EC2 instances
  hosts: instances
  become: yes
  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes
    - name: Update index.html file
      ansible.builtin.copy:
        content: "<h1>Hello from {{ inventory_hostname }}</h1>"
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
    - name: Start and Enable Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Now, run the playbook with:

```
ansible-playbook -i hosts.ini setup.yml
```

Your output should look similar to this. Changed/ok statuses signifies it ran successfully without errors.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*eLKiL-i_PhjeRUJhcf1uAg.png)

Now when you visit each instances public_ipv4_addresses, you should get something similar to this.

![captionless image](https://miro.medium.com/v2/resize:fit:1080/format:webp/1*IPvMuj-PcbQXsNUZXDBpJQ.png)

Step 3: Setup an Application Load Balancer (ALB)
------------------------------------------------

Instead of hitting each server separately, we’ll place an ALB in front of them.

To do that, navigate to **EC2** -> **Load Balancers** -> **Create Load Balancer**.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*vdKuGPvJukCSqDbVSwuqfA.png)

Select ‘**Application Load Balancer**’ and then click ‘**Create**’ button to proceed.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*DjugMzxXvDKFtu9FAyMQZA.png)

In the ‘**Basic configuration**’ section, set a unique ‘**Load balancer name**’ , select ‘**Internet-facing**’ option as the Scheme and then ‘**IPv4**’ option for IP address type as shown below.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*3HhEhBASxfYgI7mi6e8Isg.png)

In the ‘Network mapping’ section, select the whatever ‘**Availability Zones and subnets**’ that was selected previously when setting up each instances earlier on.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*YFilyDt1vic5XVQQG9FsFQ.png)

Same thing for the ‘**Security groups’.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*0yZNRVHNJpApDbhYWPUmJw.png)

For the listeners, ensure the ‘HTTP’ protocol and port ‘80’ is used.

Select ‘**Forward to target groups**’ for the ‘**Routing action’** and then click on the ‘**create target group**’. This might open up a new page in a different tab so bare this mind.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*XFttWElxmRB99To7Bxm3jg.png)

On the ‘Create target group’ page, select the options as described below:

*   **Target type**: instances

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*5xXb8qx1reNGCsOwIWlljQ.png)

*   **Target group name:** any unique name
*   **Protocol and port**: HTTP and 80
*   **IP address type**: IPv4
*   **Protocol version**: HTTP1

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*9UcXVoBN51Vt5L-PhCdCDw.png)

Click on ‘**next**’ button and on the next page, select the two instances we created previously and then click ‘**Include as pending below**’.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*14GWr2tuYSplA9DDMSY0Ng.png)

Finally, click **‘Create target group**’ to create the target group with the selected instances.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*jHYXJIv8b8I8d1288KNrZg.png)

You should see this once the process has finished.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*MGujy6ki2LSEAZ6Vr9SmIw.png)

Now go back to the previous page/tab (where we were setting up the load balancer), and click the refresh button just beside the ‘**Target group**’ select field.

You should now see the newly created target group.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*NMCQE3wfCuGoADaBu2bQWA.png)

Now, after reviewing everything and making sure its correct, click on ‘Create load balancer’ to start the creation process.

This would take a while to provision but your page should look something like this. You’ll know its ready once the ‘**Status**’ goes from ‘Provisioning’ to ‘Active’.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*pTVNIcGrHRvhkJucKNv3GQ.png)

Copy the ‘**DNS name**’ and then visit it in your browser. In this case, that should be something like ‘http://ec2-alb-435792316.us-east-1.elb.amazonaws.com’

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*oCyDmHq33L26GkuWwh67rg.png)

Awesome, now we have a working load balancer.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*PmQDQ_0GJiRYfXSNvrSD9g.png)

Refresh the page enough times and you should hit the other instance as well .

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TSq26el4_GupqKVMBr-CKg.png)

Step 4: Deploy an HTML file to S3
---------------------------------

From the console navigate to **Storage** -> **S3** and then click ‘**Create Bucket**’.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*5rHpVwyysNI1lhtFPeetew.png)

Set a globally unique **Bucket name** and disable **Block Public Access settings for this bucket** as shown below and click **Create bucket** at the bottom of the page.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ZvcxH8PTK3KsbZj-sABUsA.png)

Once its created, click **View details** to open the bucket page.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*tDCyIEnhcH6aMOXI6WRs8g.png)

Click on **Upload** to add your index.html (and other necessary files) to the bucket.

_Link to the index.html file I uploaded —_ [_https://github.com/codelikesuraj/alb-ec2-s3-tutorial_](https://github.com/codelikesuraj/alb-ec2-s3-tutorial)

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*D-XvGxHz6bPkABexNDko1w.png)

After successfully upload your file(s), go to the **Properties** section and scroll to **Static website hosting** and then click on **Edit.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ruun-7N5iUcmZ2ovExaMEw.png)

Enable it and select ‘Host a static website’ for the **Hosting type** and then **Save changes.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*2J3r3ke9-Rs_btNNys3Wzg.png)

Now, AWS will give you a **Bucket website endpoint** like this **http://s3-oct-2025.s3-website-us-east-1.amazonaws.com.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*bHqIYiGQmnNMvAI3XDJ2-A.png)

If you attempted to visit that directly, you would be getting an error like this

![captionless image](https://miro.medium.com/v2/resize:fit:1370/format:webp/1*SiUSPWVEOudL_u3MoeYukA.png)

This is because by default, S3 objects are private. We need to allow public read access.

Now, go the **Permissions** tab and click **Edit** in the **Bucket policy** section.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*UfdZEps7mTWIR6EEgqF7WQ.png)

Paste this into the **Policy** field and then **Save changes.**

```
{
  "Version": "2012-10-17",
  "Statement": [    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::s3-oct-2025/*"
    }
  ]
}
```

Note: replace the ‘arn:aws:s3:::s3-oct-2025’ with your **Bucket ARN**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*WEK4uPWwygRwlJ4TG23LJw.png)

Revisit the **Bucket website endpoint** and your website should be accessible now.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*SEYTf2jvCiWGDPiv1pd0Yg.png)

Conclusion
----------

This hands-on projected demonstrated two fundament hosting setup on AWS:

*   With EC2 + ALB, you get full control of the server environment, making it ideal for dynamic applications and backends. However, this comes with management overhead and costs since you’re running compute resources.
*   With S3 static website hosting, you get a lightweight, cost-effective, and infinitely scalable way to serve static sites. Perfect for documentation, portfolios, or frontend projects that don’t need server-side logic.

Both approaches serve different purposes. The real skill lies in knowing when to choose one over the other.
