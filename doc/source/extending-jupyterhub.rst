Extending your JupyterHub setup
===============================

The helm chart used to install JupyterHub has a lot of options for you to tweak. This page lists some of the most common changes.

Applying configuration changes
------------------------------

The general method is:

1. Make a change to the ``config.yaml``
2. Run a helm upgrade:

     .. code-block:: bash

        helm upgrade <YOUR_RELEASE_NAME> https://github.com/jupyterhub/helm-chart/releases/download/v0.2/jupyterhub-0.2.tgz -f config.yaml

   Where ``<YOUR_RELEASE_NAME>`` is the parameter you passed to ``--name`` when `installing jupyterhub <setup-jupyterhub.html#install-jupyterhub>`_ with
   ``helm install``. If you don't remember it, you can probably find it by doing ``helm list``
3. Wait for the upgrade to finish, and make sure that when you do ``kubectl --namespace=<YOUR_NAMESPACE> get pod`` the hub and proxy pods are in ``Ready`` state. Your configuration change has been applied!

   .. note::

   Currently, most config changes (except for changing with user image is used) require you to manually restart the hub pod. You can do this by finding the name of the pod with ``kubectl --namespace=<YOUR_NAMESPACE> get pod`` (it's the one that stats with hub-), and doing ``kubectl --namespace=<YOUR_NAMESPACE> delete pod <hub-pod-name>``. This will be fixed soon!


Using an existing image
-----------------------

It's possible to build your jupyterhub deployment off of a pre-existing Docker image.
To do this, you need to find an existing image somewhere (such as DockerHub), and configure
your installation to use it.

For example, UC Berkeley's `Data8 Program <https://hub.docker.com/r/berkeleydsep/singleuser-data8>`_ publishes the image they are using on Dockerhub.
To instruct JupyterHub to use this image, simply add this to your ``config.yaml`` file:

    .. code-block:: yaml

       singleuser:
           image:
              name: berkeleydsep/singleuser-data8
              tag: v0.1


You can then `apply the change <#applying-configuration-changes>`_ to the config as usual.

Setting memory and CPU guarantees / limits for your users
---------------------------------------------------------

Each user on your JupyterHub gets a slice of memory and CPU to use. By default, each user is
*guaranteed* 1G of RAM, but no enforced limits. This means each user is guaranteed at minimum
1G of RAM, but no exact upper limit (this will vary based on various factors outside of your
control). We recommend most people change this!

If you want all your users to have guaranteed access to 1Gi of RAM, and no more, you can add the
following to your config.yaml and do an upgrade:

    .. code-block:: yaml

       singleuser:
           memory:
               limit: 1G
               guarantee: 1G

Kubernetes will make sure that each user will always have access to 1G of RAM, and requests for
more RAM will fail (your kernel will usually die). The guarantee is the base amount of RAM that
is guaranteed for each user, and the limit is the amount of RAM at which new memory requests
will not be granted. You can set the limit to be higher than the guarantee if you want to allow
some users to 'burst' RAM use occasionally (and use more RAM than the guarantee).

The same applies to cpu usage too! 

Extending your software stack with s2i
--------------------------------------

s2i, also known as `Source to Image <https://github.com/openshift/source-to-image>`_, lets you
quickly convert a GitHub repository into a Docker image that we can use as a
base for your JupyterHub instance. Anything inside the GitHub repository
will exist in a user’s environment when they join your JupyterHub. If you
include a ``requirements.txt`` file in the root level of your of the repository,
s2i will ``pip install`` each of these packages into the Docker image to be
built. Below we’ll cover how to use s2i to generate a Docker image and how to
configure JupyterHub to build off of this image.

.. note::
       For this section, you’ll need to install s2i and docker.


1. **Download s2i.** This is easily done with homebrew on a mac. For linux and
   Windows it entails a couple of quick commands that you can find in the
   links below:

       - On OSX: ``brew install s2i``
       - On Linux and Windows: `follow these instructions
         <https://github.com/openshift/source-to-image#installation>`_

2. **Download and start Docker.** You can do this by downloading and installing
   Docker at `this link <https://store.docker.com/search?offering=community&platform=desktop%2Cserver&q=&type=edition>`_.
   Once you’ve started Docker, it will show up as a tiny background application.

3. **Create (or find) a GitHub repository you want to use.** This repo should
   have all materials that you want your users to access. In addition you can
   include a ``requirements.txt`` file that has one package per line. These
   packages should be listed in the same way that you’d install them using
   ``pip install``. You should also specify the versions explicitly so the image is
   fully reproducible. E.g.:

   .. code-block:: bash

          numpy==1.12.1
          scipy==0.19.0
          matplotlib==2.0

4. **Use s2i to build your Docker image.** `s2i` uses a template in order to
   know how to create the Docker image. We have provided one at the url in the
   commands below. Run this command::

       s2i build <git-repo-url>  jupyterhub/singleuser-builder:v0.1.1 gcr.io/<project-name>/<name-of-image>:<tag>

   this effectively says *s2i, build `<this repository>` to a Docker image by
   using `<this template>` and call the image `<this>`*

  .. note::
         - The project name should match your google cloud project's name.
         - Don’t use underscores in your image name. Other than this it can be
           anything memorable. This is a bug that will be fixed soon.
         - The tag should be the first 6 characters of the SHA in the GitHub
           commit for the image to build from.

5. **Push our newly-built Docker image to the cloud.** You can either push this
   to Docker Hub, or to the gcloud docker repository. Here we’ll push to the
   gcloud repository::

       gcloud docker -- push gcr.io/<project-name>/<image-name>:<tag>

6.  **Edit the JupyterHub configuration to build from this image.** We do this by editing the ``config.yaml`` file that we originally created to include the jupyter hashes. Edit ``config.yaml`` by including these lines in it:

    .. code-block:: bash

          singleuser:
            image:
              name: gcr.io/<project-name>/<image-name>
              tag: <tag>

7. **Tell helm to update JupyterHub to use this configuration.** Using the normal method to `apply the change <#applying-configuration-changes>`_ to the config.
8. **Restart your notebook if you are already logging in** If you already have a running JupyterHub session, you’ll need to restart it (by stopping and starting your session from the control panel in the top right). New users won’t have to do this.
9. **Enjoy your new computing environment!** You should now have a live computing environment built off of the Docker image we’ve created.

Authenticating with OAuth2
--------------------------

JupyterHub's `oauthenticator <https://github.com/jupyterhub/oauthenticator>`_ has support for enabling your users to authenticate via a third-party OAuth provider, including GitHub, Google, and CILogon.

Follow the service-specific instructions linked on the `oauthenticator repository <https://github.com/jupyterhub/oauthenticator>`_ to generate your JupyterHub instance's OAuth2 client ID and client secret. Then declare the values in the helm chart (``config.yaml``).

Here are example configurations for two common authentication services. Note that
in each case, you need to get the authentication credential information before
you can configure the helmchart for authentication.

**Google**

For more information see the full example of Google OAuth2 in the next section.

.. code-block:: yaml

    auth:
      type: google
      google:
        clientId: "yourlongclientidstring.apps.googleusercontent.com"
        clientSecret: "adifferentlongstring"
        callbackUrl: "http://<your_jupyterhub_host>/hub/oauth_callback"
        hostedDomain: "youruniversity.edu"
        loginService: "Your University"

**GitHub**

.. code-block:: yaml

      auth:
        type: github
        github:
          clientId: "y0urg1thubc1ient1d"
          clientSecret: "an0ther1ongs3cretstr1ng"
          callbackUrl: "http://<your_jupyterhub_host>/hub/oauth_callback"

Full Example of Google OAuth2
-----------------------------

If your institution is a `G Suite customer <https://gsuite.google.com>`_ that integrates with Google services such as Gmail, Calendar, and Drive, you can authenticate users to your JupyterHub using Google for authentication.

.. note::
       Google requires that you specify a fully qualified domain name for your hub rather than an IP address.

1. Log in to the `Google API Console <https://console.developers.google.com>`_.

2. Select a project > Create a project... and set 'Project name'. This is a short term that is only displayed in the console. If you have already created a project you may skip this step.

3. Type "Credentials" in the search field at the top and click to access the Credentials API.

4. Click "Create credentials", then "OAuth client ID". Choose "Application type" > "Web application".

5. Enter a name for your JupyterHub instance. You can give it a descriptive name or set it to be the hub's hostname.

6. Set "Authorized JavaScript origins" to be your hub's URL.

7. Set "Authorized redirect URIs" to be your hub's URL followed by "/hub/oauth_callback". For example http://example.com/hub/oauth_callback.

8. When you click "Create", the console will generate and display a Client ID and Client Secret. Save these values.

9. Type "consent screen" in the search field at the top and click to access the OAuth consent screen. Here you will customize what your users see when they login to your JupyterHub instance for the first time. Click Save when you are done.

10. In your helm chart, create a stanza that contains these OAuth fields:

.. code-block:: bash

    auth:
      type: google
      google:
        clientId: "yourlongclientidstring.apps.googleusercontent.com"
        clientSecret: "adifferentlongstring"
        callbackUrl: "http://<your_jupyterhub_host>/hub/oauth_callback"
        hostedDomain: "youruniversity.edu"
        loginService: "Your University"

The 'callbackUrl' key is set to the authorized redirect URI you specified earlier. Set 'hostedDomain' to your institution's domain name. The value of 'loginService' is a descriptive term for your institution that reminds your users which account they are using to login.
