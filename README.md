# Business Metrics with Spring Boot og CloudWatch & Terraform

## Prerequisites

To refresh your understanding of Terraform modules, take a look at the [Terraform S3 Website exercise](https://github.com/glennbechdevops/terraform-s3-website).

## Overview

The application in this repository is a "mock" bank application that features various endpoints for common banking transactions.
In this exercise, you will learn how to instrument a Spring Boot application with business metrics. By business metrics, we refer to metrics that do not pertain to the technical performance of the application, but rather to insights into user behavior, such as the amount of money transferred and the number of transactions performed.
This exercise utilizes Java, Spring Boot, and a metrics framework called Micrometer.
We will also explore how to visualize metrics in AWS CloudWatch and how to use Terraform to create a dashboard.

# Prepare your GitHub Codespaces environment

## Fork and Open in Codespaces

1. Fork this repository to your own GitHub account by clicking the "Fork" button at the top right of the repository page
2. Once forked, navigate to your forked repository
3. Click the green "Code" button and select "Codespaces"
4. Click "Create codespace on main" to launch a new Codespaces environment

GitHub Codespaces will automatically set up your development environment with all necessary tools including:
- Java 17
- Maven
- Terraform
- AWS CLI
- jq

The setup process may take a few minutes the first time you create the codespace. Once complete, you'll have a fully configured environment ready to go! 

## Verify Your Environment

The devcontainer should have already installed all necessary tools. You can verify the installations by running:

```sh
java -version    # Should show Java 17
mvn -version     # Should show Maven
terraform --version
aws --version
jq --version
```

## Create IAM Access Keys

Before you can use AWS services from your Codespace, you need to create access keys for your IAM user:

1. **Find Your IAM User:**
   - You already have an IAM user named `seat` + a number (e.g., `seat01`, `seat02`)
   - This user already has the necessary permissions

2. **Create Access Keys:**
   - Go to AWS Console → IAM → Users
   - Find and click on your user (e.g., `seat01`)
   - Go to "Security credentials" tab
   - Scroll to "Access keys" section
   - Click "Create access key"
   - Choose "Application running outside AWS"
   - Click "Next" and "Create access key"
   - **Important:** Save both the Access Key ID and Secret Access Key somewhere safe - you won't see the secret again!
   - **Keep these credentials handy - you'll need them later for the optional GitHub Actions exercise**

## Run aws configure

Now configure the AWS CLI with your credentials:

```bash
aws configure
```

When prompted, enter:
* **AWS Access Key ID**: (paste the Access Key ID you just created)
* **AWS Secret Access Key**: (paste the Secret Access Key you just created)
* **Default region name**: `eu-west-1`
* **Default output format**: `json` 

## PART 1-  Use Terraform to Create a CloudWatch Dashboard

* You should now have this repository open in your Codespaces environment
* Look in the "infra" directory - here you will find the file dashboard.tf which contains Terraform code for a CloudWatch Dashboard.
* As you can see, the dashboard is described in a JSON format. Here you can find documentation https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/CloudWatch-Dashboard-Body-Structure.html
Here you also see how one often includes text or code using "Heredoc" syntax in Terraform code, so that we don't have to worry about "newline", "Escaping" of special characters, etc. (https://developer.hashicorp.com/terraform/language/expressions/strings)

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = var.student_name
  dashboard_body = <<DEATHSTAR
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [
            "${var.student_name}",
            "account_count.value"
          ]
        ],
        "period": 300,
        "stat": "Maximum",
        "region": "eu-west-1",
        "title": "Total number of accounts"
      }
    }
  ]
}
DEATHSTAR
}
```
## Task

Run the following commands:

```bash
cd infra
terraform init
terraform plan
terraform apply
```

* You have to type in a student name, why?
* Can you think of at least two ways to fix that- so that you don't have to type a student name on plan and apply?

## Look at the Spring Boot application

* the Java application can be found in the `src/` folder

Open *BankAccountController.Java* , You'll find this code

```java
    @Override
    public void onApplicationEvent(ApplicationReadyEvent applicationReadyEvent) {
        Gauge.builder("account_count", theBank,
                b -> b.values().size()).register(meterRegistry);
    }
```

This creates a new metric - of the type Gauge, which constantly reports how many bank accounts exist in the system by 
counting the size of the values in the map holding the accounts. 

## Modify the MetricsConfig Class

You need to modify the MetricsConfig class and use your own name for the cloudwatch namespace, replace the empty string "" with your student name.  

````java
 return new CloudWatchConfig() {
        private Map<String, String> configuration = Map.of(
                "cloudwatch.namespace", "",
                "cloudwatch.step", Duration.ofSeconds(5).toString());
        
        ....
    };
````

## Start Spring Boot applikasjonen 


From the root folder  (where you cloned repository for this exercise); Start the Spring boot app with maven with

```
mvn spring-boot:run
```

The code in this repository exposes a REST interface at http://localhost:8080/account

## Test the API


### Using Codespaces Port Forwarding

When you run the Spring Boot application on port 8080, GitHub Codespaces will automatically detect it and forward the port.
You'll see a notification in VS Code, and the port will appear in the "Ports" tab. You can:
- Click on the forwarded address to open it in your browser
- Use tools like Postman or curl from your local machine by using the forwarded URL
- The forwarded URL will look something like: `https://username-repo-xxxxx.github.dev`

![](img/postman.png)

### Use Curl in the Codespaces Terminal

Curl is a command-line tool used to transfer data to or from a server, supporting a wide range of protocols including HTTP, HTTPS, FTP, and more. It is widely used for testing, sending requests, and interacting with APIs directly
fro  the terminal or in scripts.

Create an account with an id and balance
```sh
curl --location --request POST 'http://localhost:8080/account' \
--header 'Content-Type: application/json' \
--data-raw '{
    "id": 3,
    "balance" : "100000"
}'|jq
```

* See information about an account

```sh 
  curl --location --request GET 'http://localhost:8080/account/3' \
  --header 'Content-Type: application/json'|jq
```

* Transfer money from one account to another  (It will create accounts if they do not exist)

```sh
curl --location --request POST 'http://localhost:8080/account/2/transfer/3' \
--header 'Content-Type: application/json' \
--data-raw '{
    "fromCountry": "SE",
    "toCountry" : "US",
    "amount" : 500
}
'|jq
```

## Check that we see data in the dashboard

* Go to the AWS UI, and select the CloudWatch service. Choose "Dashboards".
* Search for your own student name and open the dashboard you created.
* See that you get data points on the graph.

It should look something like this:

![Alt text](img/dashboard.png  "a title")



## Do not move on unless you fully understand what has been done so far 

* Look at the micrometer documentation-  (https://docs.spring.io/spring-boot/reference/actuator/metrics.html)
* Try to add more metrics to the code, also make new endpoints if you want more code to test
* Make sure you understand the CloudWatch dashboard Syntax, at least at a high level - try to add another widget!



# PART 2

## Gauge for the Bank's Total Sum

You are now going to create a Micrometer Gauge that displays the net balance of the bank. How much money that is in all accounts - Place it inside the `public void onApplicationEvent(ApplicationReadyEvent applicationReadyEvent)` method in BankAcountController class. 

Please note that you have to import the BigDecimal class. When you add this code. Add the `import java.math.BigDecimal` line to the top of the file.

```
// This meter type "Gauge" reports the total amount of money in the bank
Gauge.builder("bank_sum", theBank,
                b -> b.values()
                        .stream()
                        .map(Account::getBalance)
                        .mapToDouble(BigDecimal::doubleValue)
                        .sum())
        .register(meterRegistry);
```

## Create a New Widget in the CloudWatch Dashboard

Extend the Terraform code in the infra folder, so that it displays an additional widget for the  metric `bank_sum` you just created. 
In the resource "aws_cloudwatch_dashboard" "main"`, in the file infra/dashboards.tf, add more code. 

```
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = var.student_name
  dashboard_body = <<DEATHSTAR
{
  "widgets": [
    {
       <<<<<< ----- INSERT ANOTHER WIDGET HERE!
    },
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [
            "${var.student_name}",
            "account_count.value"
          ]
        ],
        "period": 300,
        "stat": "Maximum",
        "region": "eu-west-1",
        "title": "Total number of accounts"
      }
    }
  ]
}
DEATHSTAR
}
```

Remember to change the X/Y values so they do not overlap!

## PART 2 - Cloudwatch Alarm
We will create an Alarm that is triggered if the total sum of the bank exceeds a given amount.

This can be done using CloudWatch. We will also create a module for this alarm, so others can benefit from it.

We will also use the service SNS, Simple Notification Service. By sending a message to an SNS topic when the alarm is triggered, we can respond to such a message, for example, by sending an email, running a lambda function, etc.

## Create Terraform Module
We will now create a Terraform module. While we work on it, it's smart to keep it on a local filesystem so we do not need to do git add/commit/push etc., to update the code.

1. Create a new folder under infra/ called _alarm_module_  
2. In this folder, create a new Terraform file named main.tf

Take not of the fact that we're following best practices, and adding a prefix that we'll use in naming resources. This way, more than one 
instance of this module can be used in same AWS Environment

```shell
resource "aws_cloudwatch_metric_alarm" "threshold" {
  alarm_name  = "${var.prefix}-threshold"
  namespace   = var.prefix
  metric_name = "bank_sum.value"

  comparison_operator = "GreaterThanThreshold"
  threshold           = var.threshold
  evaluation_periods  = "2"
  period              = "60"
  statistic           = "Maximum"

  alarm_description = "This alarm goes off as soon as the total amount of money in the bank exceeds an amount."
  alarm_actions     = [aws_sns_topic.user_updates.arn]
}

resource "aws_sns_topic" "user_updates" {
  name = "${var.prefix}-alarm-topic"
}

resource "aws_sns_topic_subscription" "user_updates_sqs_target" {
  topic_arn = aws_sns_topic.user_updates.arn
  protocol  = "email"
  endpoint  = var.alarm_email
}
```

### A Little Explanation About the aws_cloudwatch_metric_alarm Resource

- **Namespace** is typically your student name. It's the same value that you changed in the MetricsConfig.java file.
- There are a wide range of `comparison_operator` options to choose from!
- `evaluation_periods` and `period` work together to avoid the alarm being triggered by short-term "spikes" or outlier observations.
- `statistic` is an operation that is performed on all values within a time interval given by `period` - for a `Gauge` metric, in this case, we choose Maximum.
- Notice how one `resource` refers to another in Terraform!
- Terraform creates both an SNS Topic and an email subscription.

## Create a new file in the /infra/alarm_module directory, `variables.tf`

This will be the variables available in the module, as you can see, we make a sensible _default_ for threshold, but require the user of the module of specify email and prefix

```shell

variable "threshold" {
  default = "50"
  type = string
}

variable "alarm_email" {
  type = string
}

variable "prefix" {
  type = string
}

```

## Create a new file in /infra/alarm_module, outputs.tf

```
output "alarm_arn" {
  value = aws_sns_topic.user_updates.arn
}
```

You can now modify dashboard.tf in the /infra directory to include your module. It will then look like this:

```shell
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = var.student_name
  dashboard_body = <<DASHBOARD
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [
            "${var.student_name}",
            "account_count.value"
          ]
        ],
        "period": 300,
        "stat": "Maximum",
        "region": "eu-west-1",
        "title": "Total number of accounts"
      }
    }
  ]
}
DASHBOARD
}


module "alarm" { << ----------- THIS IS ADDED!
  source = "./alarm_module"
  alarm_email = var.alarm_email
  prefix = var.student_name
}

```
## Finally, you must change variables.tf in the /infra folder, and add the variable. 

When we run terraform frmo the *main* folder - we want the user to specify the email recipient for the alarm. 
*do not add* add your student email in the variable like this! 

```hcl
variable "glenn.bech@gmail.com" {
    type = string
}
```

The *name* of the  variable is alarm_email, the *value* is your email address and will be provided as a value when terraform run. You *can* Feel free to set your own email address as the default value for this variable 

```hcl
variable "alarm_email" {
    type = string
}
```


# Run the Terraform Code from Codespaces

Run the following commands:

```bash
cd infra
terraform init
terraform apply
```

## Confirm Email

For SNS to be allowed to send you emails, you must confirm your email address. You will receive an email with a link you must click the first time you run terraform apply.

## Manually Test the Alarm and Email Sending Using SNS
* Go to the AWS console
* Go to SNS
* From the left menu, select "Topics"
* Find your own Topic
* Test sending an email by pressing "Publish message" at the top right of the page

## Trigger the Alarm!

Try to create new accounts, or a new account, so that the bank's total sum exceeds 1 MNOK.

```shell
curl --location --request POST 'http://localhost:8080/account' \
--header 'Content-Type: application/json' \
--data-raw '{
    "id": 999,
    "balance" : "5000000"
}'|jq

```
- Check that the alarm goes off by seeing that you have received an email.
- Go to CloudWatch Alarms in AWS and see that the alarm's state is `IN_ALARM`.
- Get the bank's balance back to 0, for example by creating an account with a negative balance.
- See that the alarm's state moves away from `IN_ALARM`.

---

# Optional Challenges

Want more practice? Try these challenges to deepen your understanding:

## Easy Challenge: Add Transfer Counter Widget

Add a widget to your CloudWatch dashboard to visualize the `transfer` counter metric that already exists in the code.

**Hints:**
- Look at the existing widget in `dashboard.tf` as a template
- The metric name is `transfer`
- Remember to adjust x/y coordinates so widgets don't overlap
- Counters work well with the "Sum" statistic

## Medium Challenge: Histogram Metric for Transaction Amounts

Create a histogram metric to track the distribution of transaction amounts and visualize percentiles (P50, P90, P99).

**Hints:**
- Use Micrometer's `Timer` or `DistributionSummary`
- Look at the `@Timed` annotation already used in the code
- Percentiles help you understand outliers vs typical transactions
- CloudWatch can display multiple percentiles in one widget

## Hard Challenge: Currency Conversion with Tagged Metrics

Implement a new `/convert` endpoint that converts amounts between currencies and tracks metrics by currency pair.

**Hints:**
- Add tags to your metrics: `.tag("fromCurrency", from).tag("toCurrency", to)`
- Use a simple fixed exchange rate (e.g., 1 USD = 10 NOK)
- Track conversion volume by currency pair
- Create a dashboard widget that shows conversions grouped by currency

---

## Optional Exercise: Create a GitHub Actions workflow

Based on previous labs and what you have learned about Terraform state, you can optionally create a GitHub Actions workflow that applies the infrastructure automatically on pushes to the main branch.

**Need inspiration?** You can peek at a working example pipeline in the [Terraform S3 Website repository - Part 2](https://github.com/glennbechdevops/terraform-s3-website?tab=readme-ov-file#part-2-avansert-terraform---moduler-remote-state-og-cicd).

### Add AWS Credentials to GitHub Secrets

You already created AWS access keys earlier in this exercise. Now you need to add them to GitHub Secrets:

1. **Go to your GitHub repository**
2. **Click Settings → Secrets and variables → Actions**
3. **Click "New repository secret"**
4. **Add two secrets:**
   - Name: `AWS_ACCESS_KEY_ID`, Value: (the Access Key ID you saved earlier)
   - Name: `AWS_SECRET_ACCESS_KEY`, Value: (the Secret Access Key you saved earlier)

### Workflow Tasks

* Create a `.github/workflows` directory with a Terraform workflow file
* Configure the workflow to run `terraform plan` on pull requests and add plan output as a PR comment
* Configure the workflow to run `terraform apply` on pushes to main branch
* Use the GitHub Secrets you created above for AWS credentials
* Configure Terraform state properly using AWS S3 as a backend (with DynamoDB state locking if you're feeling adventurous!) 


