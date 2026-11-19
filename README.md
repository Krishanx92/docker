# WSO2 Dockerfiles

Dockerfiles for building the Docker images of WSO2 products yourself, for example
on a different base image.

## Repository structure

Dockerfiles are organized by product category, product, product version, and
base operating system:

```
<product category>/
└── <product>/
    ├── README.md
    └── <product version>/
        └── <base OS>/
            └── Dockerfile
```

Each product's `README.md` explains how to build its images.

## Supported versions

Dockerfiles are maintained for the two most recent versions of each product.

## License

Licensed under the [Apache License 2.0](LICENSE).
