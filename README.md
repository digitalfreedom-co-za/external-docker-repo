# DigitalFreedom External Docker Repo

Public container registry for DigitalFreedom customers. It contains only released, versioned images of selected DigitalFreedom applications that are offered to customers. Snapshot, development and test builds are never published here and stay in the internal registry.

## License

The images in this registry may only be used with a valid license for the respective application from DigitalFreedom Global LLC. Pulling, running, copying or distributing an image without the corresponding license is not permitted. Public availability of an image does not grant any right of use.

## Registry

```
ghcr.io/digitalfreedom-co-za/external-docker-repo/<service>:<version>
```

## Tags

| Tag | Meaning |
| --- | --- |
| `1.2.0` | Exact release, recommended for production |
| `1.2` | Latest patch of a minor release |
| `latest` | Latest release |

Services that support several databases publish one image per database with a suffix, for example `1.2.0-postgresql` and `1.2.0-oracle`.

## Usage

```
docker pull ghcr.io/digitalfreedom-co-za/external-docker-repo/<service>:<version>
```

Each application documents its environment variables and a Docker Compose example in its own documentation. Always pin an exact version in production.

## Support

DigitalFreedom Global LLC, https://digitalfreedom.co.za
