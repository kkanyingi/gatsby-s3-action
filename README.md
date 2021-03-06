# gatsby-s3-action

**Deploy a Gatsby site to an AWS S3 bucket and optionally invalidate a CloudFront distribution**

- Copies a Gatsby site to the root of an S3 bucket (uses `sync` so old files in the bucket will be removed).
- Sets cache headers as defined by the rules described in the [Gatsby documentation](https://www.gatsbyjs.org/docs/caching/).
- Fast - uses AWS Cli commands for mass file operations which only create/modify files as needed.
- Suitable for hosting with or without CloudFront. If a CloudFront distribution is specified then it will be invalidated after deployment.

Please review the associated [GitHub Actions powered Gatsby AWS how-to guide](https://blog.elantha.com/gatsby-s3-cloudfront/) if you're interested in setting up a Gatsby site on AWS from scratch.

## S3 Static Website Hosting

This section describes the use of this action if you're hosting directly from an S3 bucket using **S3 Static Website Hosting** (without CloudFront). If you're using CloudFront please see the section below.

### Typical workflow yaml file:

```yml
name: Deploy

on:
  push:
    branches:
      - master
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Use Node.js 12
        uses: actions/setup-node@v1
        with:
          node-version: 12.x
      - name: Build
        run: |
          npm ci
          npm run build
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-2
      - name: Deploy
        uses: jonelantha/gatsby-s3-action@v1
        with:
          dest-s3-bucket: your_bucket
          public-source-path: your_gatsby_public_directory
```
### Notes:

- `your_bucket` should be changed to the name of your bucket
- `your_gatsby_public_directory` optionally set the path to your gatsby ./public directory (default: ./public/)
- You'll need to [setup an AWS IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) with `Programmatic Access` and then configure the `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in the Settings/Secrets area of the repo. Ideally you should follow [Amazon IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) and grant least privileges to the user:
  - `s3:ListBucket` on `arn:aws:s3:::your_bucket`
  - `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` on `arn:aws:s3:::your_bucket/*`
- For the S3 Static Website Hosting settings:
  - Index Document should be `index.html`
  - Error Document should be `404.html`
- If you plan to use Gatsby redirects you'll need to use a Gatsby redirect plugin such as one of the following:
  - [gatsby-plugin-client-side-redirect](https://www.gatsbyjs.org/packages/gatsby-plugin-client-side-redirect/)
  - [gatsby-plugin-meta-redirect](https://www.gatsbyjs.org/packages/gatsby-plugin-meta-redirect/)


## S3 / CloudFront hosting

This section describes usage for when your Gatsby site is stored on S3 and a served via a CloudFront distribution. If you're new to AWS you may also be interested in reviewing the [GitHub Actions powered Gatsby AWS how-to guide](https://blog.elantha.com/gatsby-s3-cloudfront/).


### Typical workflow yaml file:

```yml
name: Deploy

on:
  push:
    branches:
      - master
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Use Node.js 12
        uses: actions/setup-node@v1
        with:
          node-version: 12.x
      - name: Build
        run: |
          npm ci
          npm run build
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-2
      - name: Deploy
        uses: jonelantha/gatsby-s3-action@v1
        with:
          dest-s3-bucket: your_bucket
          cloudfront-id-to-invalidate: YOURCLOUDFRONTID
```
### Notes:

- `your_bucket` should be changed to the name of your bucket
- `YOURCLOUDFRONTID` should be changed to the ID of your CloudFront distribution
- You'll need to [setup an AWS IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) with `Programmatic Access` and then configure the `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in the Settings/Secrets area of the repo. Ideally you should follow [Amazon IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) and grant least privileges to the user:
  - `s3:ListBucket` on the `arn:aws:s3:::your_bucket`
  - `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` on `arn:aws:s3:::your_bucket/*`
  - `cloudfront:CreateInvalidation` on `arn:aws:cloudfront::YOURACCOUNT_ID:distribution/YOURCLOUDFRONTID`

  _For a complete walkthrough of setting up the user, please see the first section of the associated [Setting up Github Actions for Gatsby](https://blog.elantha.com/gatsby-github-actions/) guide_
- The S3 bucket does not need to be set for public access. The CloudFront distribution should have access to your bucket (this is easy to configure when setting up the CloudFront distribution).
- You will need to setup a [lambda@edge](https://aws.amazon.com/lambda/edge/) function on the CloudFront distribution to properly handle serving up `index.html` files, more information in [this guide](https://blog.elantha.com/cloudfront-index-lambda/)
- If you plan to use Gatsby redirects you'll need to use a Gatsby redirect plugin such as one of the following:
  - [gatsby-plugin-client-side-redirect](https://www.gatsbyjs.org/packages/gatsby-plugin-client-side-redirect/)
  - [gatsby-plugin-meta-redirect](https://www.gatsbyjs.org/packages/gatsby-plugin-meta-redirect/)

## Action Parameters Reference

| Argument | Status | Description |
|--------|-------|------------|
| dest-s3-bucket | Required | The destination S3 Bucket |
| browser-cache-duration | Optional | The cache duration (in seconds) to instruct browsers to cache files for. This is only for files which should be cached as per [Gatsby caching recommendations](https://www.gatsbyjs.org/docs/caching/). Default is 31536000 (1 year) |
| cdn-cache-duration | Optional | The cache duration (in seconds) to instruct a CDN (if there is one) to cache files for. If on a development environment and you want to avoid issuing CloudFront invalidations you could set this to 0. Default is 31536000 (1 year) |
| cloudfront-id-to-invalidate | Optional | The ID of the CloudFront distribution to invalidate. |