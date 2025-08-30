name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build app
        run: npm run build

      # Optionally archive build output
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: |
            build/
            dist/

  deploy:
    name: Deploy to Elastic Beanstalk
    needs: build-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup AWS CLI
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Install dependencies and build (needed for EB bundle)
        run: |
          npm ci
          npm run build

      - name: Zip application bundle
        run: |
          zip -r app.zip . -x '*.git*' 'node_modules/*' # adjust excludes as needed

      - name: Deploy to Elastic Beanstalk
        run: |
          eb init ${{ secrets.EB_APPLICATION }} --region ${{ secrets.AWS_REGION }} --platform node.js
          eb use ${{ secrets.EB_ENVIRONMENT }}
          eb deploy --staged
