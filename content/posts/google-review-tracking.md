+++
title = "Google Review Tracking"
description = "Discover an innovative, cost-effective solution for tracking customer reviews using AWS services, complete with a GitHub repo for easy implementation."
keywords = ["customer reviews", "small business", "AWS", "Lambda", "SES", "S3", "NodeJS", "Terraform", "automation", "CRM", "Google reviews"]
categories = ["technology", "small business", "automation"]
date = "2024-09-01T10:00:00Z"

+++

**TL;DR:** I've created an automated solution for tracking Google reviews using AWS services. The best part? You can implement it yourself! Check out the full code and setup instructions on GitHub: [google-review-email-processor](https://github.com/imduffy15/google-review-email-processor)

A small business owner came to me with a tricky problem: he needed a way to track which customers had left reviews for his business and which hadn't. His goal was simple—avoid bothering customers who had already left a review with additional requests. Straightforward, right? Not quite.

![](/images/google-review-tracking.png)

## The Initial Game Plan: APIs and Dead Ends

I started by checking out the APIs Google offers. I mean, that's where all the reviews live, so it made sense, right?

- **Google Maps API**: Looked promising, but it only gives a handful of reviews. No dice.

- **Google Business API**: This is application only. I applied and got rejected. Bummer.

I even toyed with the idea of web scraping, but quickly decided it wasn't a good idea. Too expensive, too fiddly to maintain, and let's be real – Google probably wouldn't be too thrilled about it.

## The Lightbulb Moment: Email to the Rescue!

Just when I was about to throw in the towel, *ping* – an email notification for a new review popped up. And that's when it hit me.

Google sends out an email every single time someone leaves a review for your business. And guess what? They have all the data I needed:

- A snippet of the review
- The reviewer's name
- The star rating

Bingo! I had found my data source.

## The Solution: Putting the Pieces Together

1. **Email Forwarding Magic**: Set up a rule to bounce those Google review notifications over to an AWS SES address.
2. **AWS SES**: Receives the email and stores it to S3.
3. **Lambda Function**:
    - Listen for new emails on S3.
    - Parse the reviewer name, review snippet, and rating.
    - Do a fuzzy match on the CRM to mark the customer as having left a review.

And just like that, I had an automated system keeping the CRM up-to-date with all the review info. 

## Why I Love This Solution

- **Wallet-Friendly**: AWS SES and Lambda costs near nothing. The S3 bucket is configured to delete emails after 1 day, so the storage costs are minimal.

- **No Polling Necessary**: Google will notify us when new reviews are submitted. No need to constantly check for new reviews.

- **Hands-Off**: Once it's set up, it should just do its thing. Set it and forget it!

- **Scalable**: It should work for any size business, given Lambda's ability to scale.

- **Plays Nice**: I'm not stepping on any toes or breaking any rules. It's all above board.

## Implement It Yourself: The GitHub Repo

**GitHub Repository**: [google-review-email-processor](https://github.com/imduffy15/google-review-email-processor)

What's in the repo:

- **Google Review Email Parser**: A lambda function in NodeJS that parses the email notification from Google of new reviews

- **Terraform Code**: Terraform scripts to spin up everything.
