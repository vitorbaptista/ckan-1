====================
Doing a CKAN release
====================

This section documents the steps followed by the development team to do a
new CKAN release.

.. seealso::

   :doc:`/maintaining/upgrading/index`
     An overview of the different kinds of CKAN release, and the process for
     upgrading a CKAN site to a new version.

----------------
Process overview
----------------

The process of a new release starts with the creation of a new release branch.
A release branch is the one that will be stabilized and eventually become the actual
released version. Release branches are always named ``release-vM.m.p``, after the
:ref:`major, minor and patch versions <releases>` they include. Major and minor versions are
always branched from master. Patch releases are always branched from the most recent tip
of the previous patch release branch.

 ::

     +--+---------------------------------------+------------->  Master
        |                                       |
        +----------------->  release-v2.4.0     +---------->  release-v2.5.0
                          |
                          +--------->  release-v2.4.1
                                    |
                                    +------>  release-v2.4.2

Once a release branch has been created there is generally a three-four week period until
the actual release. During this period the branch is tested and fixes cherry-picked. The whole
process is described in the following sections.


.. _beta-release:

----------------------
Doing the beta release
----------------------

Beta releases are branched off a certain point in master and will eventually
become stable releases.

Turn this file into a github issue with a checklist using this command::

   echo 'Full instructions here: https://github.com/ckan/ckan/blob/master/doc/contributing/release-process.rst'; egrep '^(\#\.|Doing|Leading)' doc/contributing/release-process.rst | sed 's/^\([^#]\)/\n## \1/g' | sed 's/\#\./* [ ]/g' |sed 's/::/./g'

#. Create a new release branch::

        git checkout -b release-v2.5.0

   Update ``ckan/__init__.py`` to change the version number to the new version
   with a *b* after it, e.g. *2.5.0b*.
   Commit the change and push the new branch to GitHub::

        git commit -am "Update version number"
        git push origin release-v2.5.0

   You will probably need to update the same file on master to increase the
   version number, in this case ending with an *a* (for alpha).

   During the beta process, all changes to the release branch must be
   cherry-picked from master (or merged from special branches based on the
   release branch if the original branch was not compatible).

   As in the master branch, if some commits involving CSS changes are
   cherry-picked from master, the less compiling command needs to be run on
   the release branch. This will update the ``main.css`` file::

        ./bin/less --production
        git commit -am "Rebuild CSS"
        git push

   There will be a final front-end build before the actual release.

#. Update beta.ckan.org to run new branch.

   The beta staging site
   (http://beta.ckan.org, currently on s084) must be set to track the latest beta
   release branch to allow user testing. This site is automatically updated nightly.

   Check the message on the front page reflects the current version. Edit it as
   a syadmin here: http://beta.ckan.org/ckan-admin/config

#. Announce the branch and ask for help testing on beta.ckan.org on ckan-dev.

#. Make latest translation strings available on Transifex.

   During beta, a translation freeze is in place (ie no changes to the translatable
   strings are allowed). Strings need to be extracted and uploaded to
   Transifex_:

   a. Install the Babel and Transifex libraries if necessary::

        pip install --upgrade Babel
        pip install transifex-client

   b. Create a ``~/.transifexrc`` file if necessary with your login details
      (``token`` should be left blank)::

        [https://www.transifex.com]
        hostname = https://www.transifex.com
        username = <username>
        password = <password>
        token =

   c. Extract new strings from the CKAN source code into the ``ckan.pot``
      file. The pot file is a text file that contains the original,
      untranslated strings extracted from the CKAN source code.::

        python setup.py extract_messages

      The po files are text files, one for each language CKAN is translated to,
      that contain the translated strings next to the originals. Translators edit
      the po files (on Transifex) to update the translations. We never edit the
      po files locally.

   c. Get the latest translations (of the previous CKAN release) from
      Transifex, in case any have changed since.

        tx pull --all --force

   d. Delete any language files which have no strings or hardly any translated.
      Check for 5% or less on Transifex.

   e. Update the ``ckan.po`` files with the new strings from the ``ckan.pot`` file::

        python setup.py update_catalog --no-fuzzy-matching

      Any new or updated strings from the CKAN source code will get into the po
      files, and any strings in the po files that no longer exist in the source
      code will be deleted (along with their translations).

      We use the ``--no-fuzzy-matching`` option because fuzzy matching often
      causes problems with Babel and Transifex.

      If you get this error for a new translation:

          babel.core.UnknownLocaleError: unknown locale 'crh'

      then it's Transifex appears to know about new languages before Babel
      does. Just delete that translation locally - it may be ok with a newer Babel in
      later CKAN releases.

   f. Run msgfmt checks::

          find ckan/i18n/ -name "*.po"| xargs -n 1 msgfmt -c

      You must correct any errors or you will not be able to send these to Transifex.

      A common problem is that Transifex adds to the end of a po file as
      comments any extra strings it has, but msgfmt doesn't understand them. Just
      delete these lines.

   g. Run our script that checks for mistakes in the ckan.po files::

        pip install polib
        paster check-po-files ckan/i18n/*/LC_MESSAGES/ckan.po

      If the script finds any mistakes then at some point before release you
      will need to correct them, but it doesn't need to be done now, since the priority
      is to announce the call for translations.

      When it is done, you must do the correctsion on Transifex and then run
      the tx pull command again, don't edit the files directly. Repeat until the
      script finds no mistakes.

   h. Edit ``.tx/config``, on line 4 to set the Transifex 'resource' to the new
      major release name (if different), using dashes instead of dots.
      For instance v2.4.0, v2.4.1 and v2.4.2 all share: ``[ckan.2-4]``.

   i. Create a new resource in the CKAN project on Transifex by pushing the new
      pot and po files::

        tx push --source --translations --force

      Because it reads the new version number in the ``.tx/config`` file, tx will
      create a new resource on Transifex rather than updating an existing
      resource (updating an existing resource, especially with the ``--force``
      option, can result in translations being deleted from Transifex).

      If you get a 'msgfmt' error, go back to the step where msgfmt is run.

   j. On Transifex give the new resource a more friendly name. Go to the
      resource e.g. https://www.transifex.com/okfn/ckan/2-5/ and the settings are
      accessed from the triple dot icon "...". Keep the slug like "2-4", but change
      the name to be like "CKAN 2.5".

   k. Update the ``ckan.mo`` files by compiling the po files::

        python setup.py compile_catalog

      The mo files are the files that CKAN actually reads when displaying
      strings to the user.

   l. Commit all the above changes to git and push them to GitHub::

        git add ckan/i18n/*.mo ckan/i18n/*.po
        git commit -am "Update strings files before CKAN X.Y call for translations"
        git push

#. Send an annoucement email with a call for translations.

   Send an email to the ckan-dev list, tweet or post it on the blog. Make sure
   to post a link to the correct Transifex resource (like `this one
   <https://www.transifex.com/okfn/ckan/2-5/>`_) and tell users that they can
   register on Transifex to contribute. Give a deadline in two weeks time.

#. Create debian packages.

   Ideally do this once a week. Create the deb package with the latest release
   branch, using ``betaX`` iterations. Deb packages are built using Ansible_
   scripts located at the following repo:

    https://github.com/ckan/ckan-packaging

   The repository contains furhter instructions on how to run the scripts, but
   essentially you need to generate the packages (one for precise and one for
   trusty) on your local machine and upload them to the Amazon S3 bucket.

   To generate the packages, run::

     ./ckan-package -v 2.x.y -i betaX

   To upload the files to the S3 bucket, you will need the relevant credentials and
   to install the `Amazon AWS command line interface <http://docs.aws.amazon.com/cli/latest/userguide/installing.html>`_

   Make sure to upload them to the `build` folder, so they are not mistaken by
   the stable ones::

     aws s3 cp python-ckan_2.5.0-precisebeta1_amd64.deb s3://packaging.ckan.org/build/python-ckan_2.5.0-precisebeta1_amd64.deb

-------------------------
Leading up to the release
-------------------------

#. Update the CHANGELOG.txt with the new version changes.

   * Add the release date next to the version number
   * Add the following notices at the top of the release, reflecting whether
     updates in requirements, database or Solr schema are required or not::

        Note: This version requires a requirements upgrade on source installations
        Note: This version requires a database upgrade
        Note: This version does not require a Solr schema upgrade

   * Check the issue numbers on the commit messages for information about
     the changes. The following gist has a script that uses the GitHub API to
     aid in getting the merged issues between releases:

        https://gist.github.com/amercader/4ec55774b9a625e815bf

     Other helpful commands are::

        git branch -a --merged > merged-current.txt
        git branch -a --merged ckan-1.8.1 > merged-previous.txt
        diff merged-previous.txt merged-current.txt

        git log --no-merges release-v1.8.1..release-v2.0
        git shortlog --no-merges release-v1.8.1..release-v2.0

#. A week before the translations will be closed send a reminder email.

#. Once the translations are closed, sync them from Transifex.

   Pull the updated strings from Transifex, check them, compile and push as
   described in the previous steps::

        tx pull --all --force
        paster check-po-files ckan/i18n/*/LC_MESSAGES/ckan.po
        python setup.py compile_catalog
        git commit -am " Update translations from Transifex"
        git push

#. A week before the actual release, announce the upcoming release(s).

   Send an email to the
   `ckan-announce mailing list <http://lists.okfn.org/mailman/listinfo/ckan-announce>`_,
   so CKAN instance maintainers can be aware of the upcoming releases. List any
   patch releases that will be also available. Here's an `example
   <https://lists.okfn.org/pipermail/ckan-announce/2015-July/000013.html>`_ email.

-----------------------
Doing the final release
-----------------------

Once the release branch has been thoroughly tested and is stable we can do
a release.

#. Run the most thorough tests::

        nosetests ckan/tests --ckan --ckan-migration --with-pylons=test-core.ini

#. Do a final build of the front-end and commit the changes::

        paster front-end-build
        git commit -am "Rebuild front-end"

#. Review the CHANGELOG to check it is complete.

#. Check that the docs compile correctly::

        rm build/sphinx -rf
        python setup.py build_sphinx

#. Remove the beta letter in the version number.

   The version number is in ``ckan/__init__.py``
   (eg 1.1b -> 1.1) and commit the change::

        git commit -am "Update version number for release X.Y"

#. Tag the repository with the version number.

   Make sure to push it to GitHub afterwards::

        git tag -a -m '[release]: Release tag' ckan-X.Y
        git push --tags

#. Create the final deb packages.

   Using the `packaging scripts <https://github.com/ckan/ckan-packaging>`_ on your
   local machine, build the final deb packages::

     ./ckan-package -v 2.x.y -i 1

   To upload the files to the S3 bucket, you will need the relevant credentials and
   to install the `Amazon AWS command line interface <http://docs.aws.amazon.com/cli/latest/userguide/installing.html>`_

   Make sure to drop the patch version and iteration before uploading the packages::


    mv python-ckan_2.5.0-precise1_amd64.deb python-ckan_2.5-precise_amd64.deb
    mv python-ckan_2.5.0-trusty1_amd64.deb python-ckan_2.5-trusty_amd64.deb

   To upload them::

     aws s3 cp python-ckan_2.5-trusty_amd64.deb s3://packaging.ckan.org/python-ckan_2.5-trusty_amd64.deb

#. Upload the release to PyPI::

        python setup.py sdist upload

   You will need a PyPI account with admin permissions on the ckan package,
   and your credentials should be defined on a ``~/.pypirc`` file, as described
   `here <http://docs.python.org/distutils/packageindex.html#pypirc>`_
   If you make a mistake, you can always remove the release file on PyPI and
   re-upload it.

#. Enable the new version of the docs on Read the Docs.

   (You will need an admin account.)

   a. Go to the `Read The Docs`_ versions page
      and enable the relevant release (make sure to use the tag, ie ckan-X.Y,
      not the branch, ie release-vX.Y).

   b. If it is the latest stable release, set it to be the Default Version and
      check it is displayed on http://docs.ckan.org.

#. Write a CKAN blog post and announce it to ckan-announce & ckan-dev.

   CKAN blog here: <http://ckan.org/wp-admin>`_

   In the email, include the relevant bit of changelog.

#. Cherry-pick the i18n changes from the release branch onto master.

   We don't generally merge or cherry-pick release branches into master, but
   the files in ckan/i18n are an exception. These files are only ever changed
   on release branches following the :ref:`beta-release` instructions above,
   and after a release has been finalized the changes need to be cherry-picked
   onto master.

   To find out what i18n commits there are on the release-v* branch that are
   not on master, do::

     git log master..release-v* ckan/i18n

   Then ``checkout`` the master branch, do a ``git status`` and a ``git pull``
   to make sure you have the latest commits on master and no local changes.
   Then use ``git cherry-pick`` when on the master branch to cherry-pick these
   commits onto master. You should not get any merge conflicts. Run the
   ``check-po-files`` command again just to be safe, it should not report any
   problems. Run CKAN's tests, again just to be safe.  Then do ``git push
   origin master``.


.. _Transifex: https://www.transifex.com/projects/p/ckan
.. _`Read The Docs`: http://readthedocs.org/dashboard/ckan/versions/
.. _Ansible: http://ansible.com/
