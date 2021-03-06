As of 2.* versions of [`django-haystack`](https://github.com/django-haystack/django-haystack), the newest features in 5.* versions of Solr backend get `django-haystack` little bit confused. But the two can still happily work together with a few bits of extra configuration.

You just need to tell Solr to lay out the config for your Solr index in the schema-based way, without that fancy data-driven schemaless mode that is being pushed on new Solr users by default. Schemaless mode has some considerable [downsides](http://www.slideshare.net/lucenerevolution/schemaless-solr-and-the-solr-schema-rest-api/12) anyways and won't let you add for example `solr.PhoneticFilterFactory` analyzer to one of your field types as explicitly if you ever decide that you need one. Not to mention that, in the schemaless mode, all fields are suddenly multi-valued.

***
### If you are a Solr 5.5+ user and Solr is having troubles with its schema, you could try the following after completing the setup steps

It seems that there is a new ["managed schema" feature](https://cwiki.apache.org/confluence/display/solr/Managed+Schema+Definition+in+SolrConfig) being introduced. I'm not sure exactly what the implication of this is (it seems to allow the modification of schemas through an API) but it needs to be disabled, otherwise the regular `schema.xml` will not be detected, causing the above error. 

To disable, open the solr config `/usr/local/Cellar/solr/5.5.0/server/solr/MYCORE/conf/solrconfig.xml` and remove the following:

```
<!--
    ...
-->
<schemaFactory class="ManagedIndexSchemaFactory">
    <bool name="mutable">true</bool>
    <str name="managedSchemaResourceName">managed-schema</str>
</schemaFactory>
```

Alternatively, it seems that you could just replace it with:

```
<schemaFactory class="ManagedIndexSchemaFactory">
```

Optionally, you can also delete the "managed schema" config file:

    rm -r /usr/local/Cellar/solr/5.5.0/server/solr/MYCORE/conf/managed-schema

***

You can skip the first step if you already have your Solr-as-a-**service** installed.

### 1. Install Solr as a service

* Install Java if needed.
* Download `solr-<version>.tgz` from [Solr Releases](http://www.us.apache.org/dist/lucene/solr/).
* Extract the installation script from the archive and run it:

```sh
tar xzf solr-<version>.tgz solr-<version>/bin/install_solr_service.sh --strip-components=2
sudo bash ./install_solr_service.sh solr-<version>.tgz
```

* Installing Solr as a service lets it start automatically on every system boot; if required, you can restart the server with `sudo service solr restart`.

### 2. Create a schema-based Solr core

* `django-haystack` currently only supports Solr indexes based on a schema, which is stored in `schema.xml` file of a Solr core's config, so create a core from the particular schema-based configuration called "basic_configs", which comes predefined with the Solr's installation:

```sh
sudo su - solr -c '/opt/solr/bin/solr create -c <core_name> -d basic_configs'
```

* In the command above, core creation needs to be performed on behalf of `solr` user because Solr would run into permission problems otherwise.

### 3. Override `django-haystack` default template for `schema.xml`

* Grab `solr.xml` file from this repository and put it into your Django project where `django-haystack` will be able to locate the template (more on this [here](http://django-haystack.readthedocs.org/en/v2.4.0/installing_search_engines.html#solr)), such as:

```
<project_name>/templates/search_configuration/solr.xml
```

* `solr.xml` is how `django-haystack` likes to call `schema.xml` templates.
* Now you can use analyzers of your preference by modifying the template.
* You can see the `django-haystack`-specific config near the top of the template, so if you ever need to use *another* initial template, make sure to remove from your template the declarations for `<field name="id" ...` and for `<uniqueKey>` as these will be declared by the `django-haystack`-specific config and copy the `django-haystack`-specific config into the same spot in your template.

### 4. Fine-tune `django-haystack` settings

* Modify the settings for `django-haystack` in your Django settings to make it talk to a specific core:

```python
HAYSTACK_CONNECTIONS = {
    "default": {
        "ENGINE": "haystack.backends.solr_backend.SolrEngine",
        "URL": "http://127.0.0.1:8983/solr/<core_name>"
    },
    # ... other settings ...
}
```

### 5. Make it easy to rebuild Solr schema

* There is no need for manual `schema.xml` copying and Solr restarting whenever you rebuild your schema for `django-haystack` as the schema can be put directly into the core's config and the core can then be reloaded, which is way faster than restarting the entire server, all with a single command:

```sh
python manage.py build_solr_schema --filename=/var/solr/data/<core_name>/conf/schema.xml && curl 'http://localhost:8983/solr/admin/cores?action=RELOAD&core=<core_name>&wt=json&indent=true'
```

* As before, change `<core_name>` to your Solr core's name.
* You could also add ` && python manage.py rebuild_index` to the command if you've modified fields in your `SearchIndex` classes and want the changes to get reflected in the index for all model instances regardless of when they were or will be indexed.
* You could place your other Solr config files under `search_configuration`, such as `solrconfig.xml` or `synonyms.txt`, and sync it all together (with e.g. `rsync --exclude=solr.xml ...`).
* You will likely need to change the permissions on `/var/solr/data/<core_name>` to more liberal ones for the above line to execute.
