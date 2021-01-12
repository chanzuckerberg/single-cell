We want to allow the desktop and hosted cellxgene products to evolve independently.  They address pretty different use cases, so we can move faster by not requiring them to provide the same features or having overlapping implementations.

The main difference between the two products is requirements around the backends. We have one backend that runs locally and one that runs on AWS. These are the differences in requirements between the two:

| Behavior | localhost | AWS | 
| ------ | ------ | --- |
| Cache headers | No | Yes |
| Multidataset/dataroot | No? | Yes |
| CXG Adaptor | No | Yes |
| AuthN | No | Yes |
| Public/Private | No | Yes |
| Userinfo | No | Yes |
| Secrets Manager | No | Yes |
| Health | No | Yes |
| ToS/Privacy | No | Yes |
| Script Inclusion | No | Yes |
| Corpora Schema Handling | No? | Yes |
| H5AD Adaptor | Yes | No |
| Port/Host | Yes | No |
| Annotations Ontology | Yes | No |
| Reëmbedding | Yes | No |
| Obs/var Names | Yes | No |
| Local CSV Resources | Yes | No |

Currently, both backend implementations are in the `cellxgene/server` directory, and code is shared freely between them. But looking at all the code, there actually isn't all that much that both backends use:

| Path | localhost | AWS | Shared | Comment |
| --- | :---: | :---: | :---: | ----- |
| app/ | X | X |  | Defines the interface, so will vary between backends |
| auth/ | | X | | |
| cli/ | X | | | Note this also includes the conversion tools, which is part of neither backend |
| common/annotations/ | X | | | Assuming we turn off hosted annotations |
| common/config/ | X | X | | Can be pared back quite a bit in each backend |
| common/utils/ | | | X | Some can be split, but a lot of shared utility functions |
| common/corpora.py | | X
| compute/ | X | X | | There are separate algorithm implementations for h5ad and cxg |
| converters/ | X | | | `cellxgene prepare` is arguably part of localhost backend. Schema stuff is probably neither |
| data_anndata/ | X | | |
| data_cxg/ | | X | | |
| data_common/ | | | X | Actually shared code: flatbuffers and cache |
| db/ | | | | For hosted user annotations |
| eb/ | | X | | Elastic Beanstalk config/deployment |

And a lot of code that appears shared really just branches between localhost and AWS deployments. For example, consider the function that creates the response to GET requests to `/config`:

```python3
def config_get(app_config, data_adaptor):
    config = get_client_config(app_config, data_adaptor)
    return make_response(jsonify(config), HTTPStatus.OK)
```

The response is driven by values in `app_config`, which is different for localhost and AWS. So we can reorganize the backend code to cleanly separate the implementations, say a `cellxgene/local_server` and `cellxgene/hosted_backend`.



