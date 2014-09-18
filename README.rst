bkp3com
=======

Simple scripts to auto-backup configuration of the old 3com switches.

How does it work
----------------

The hardware setup consists of a **backup server** and one or more
**switches**. The backup server runs backup scripts periodically, and the
scripts download current configuration from switches. The configuration is
stored in a git repository on server. Each script commits last changes
at the end of its job.

Supported devices
-----------------

Currently supported models are:

- 4210 (3CR17334-91) - tested with ``3Com OS V3.01.12s168`` firmware,


Dependencies
------------

The following tools are required for the scripts to run

- ``sftp``,
- ``git``

Initial configuration
---------------------

#. Install dependencies, for example::

      apt-get install openssh-client git

#. Clone this repository and enter it::

      cd /tmp/
      git clone https://github.com/ptomulik/bkp3com
      cd bkp3com/

#. Copy content of ``scripts`` directory to a directory within your ``$PATH``,
   for example::

      cp -r scripts/* /usr/local/bin/

#. Create user named ``bkp3com``, with shell access and home directory
   (``/home/bkp3com`` in our examples)::

      useradd -m -c '3com auto-backup' bkp3com

#. Generate SSH keys for ``bkp3com``::

      su - bkp3com
      ssh-keygen

   For convenience the private key should not be password protected.

#. Create ``.gitconfig`` for user ``bkp3com``::

      su - bkp3com
      cat > ~/.gitconfig
      [user]
        name = 3com auto-backup
        email = admin@example.com

#. Perform necessary **Preparations** for your switches (see the section
   **Preparations** below).
#. Write a cron job (e.g. ``/etc/cron.daily/bkp3com`` to run backup scripts
   with appropriate options). In my installation I have something like this,
   for example::

        DOMAIN="mgmt.meil.pw.edu.pl"
        BKP3COM="/usr/local/bin/bkp3com-4210"
        SWITCHES="sw-01 sw-02 sw-03 sw-04"
        REPONAME=daily
        su - bkp3com -c "$BKP3COM -r $REPONAME -d $DOMAIN $SWITCHES"

   The whole backup is being stored under ``/home/bkp3com/daily/`` (its the
   path of the repo root).

Usage
-----

3com 4210
^^^^^^^^^

To trigger backup, you have to run ``bkp3com-4210``, see::

    bkp3com-4210 -h

At minimum you have to provide a list of switches to be backed up, for example::

    bkp3com-4210 sw-01 sw-02

The names ``sw-01`` and ``sw-02`` should be a host names of switches, such that
the server is able to resolve ``sw-01`` and ``sw-02`` to their corresponding IP
addresses.

If you have a group of switches in one domain, you may define the domain via
``-d`` option::

    bkp3com-4210 -d mgmt.example.com sw-01 sw-02

By default, the output is stored in a git repository under ``$HOME/bkp3com``.
The repository is created if it does not exist. Custom repository path may
be provided via ``-r`` option, e.g.::

    bkp3com-4210 -r /home/bkp3com/daily -d mgmt.example.com sw-01 sw-02

The configuration files will be written-out to subdirectories named after the
switch names, in our last example::

    /home/bkp3com/daily/sw-01/
    /home/bkp3com/daily/sw-02/

Preparations
------------

3com 4210
^^^^^^^^^

The old 4210 switches are backed up by "sftp get" method. Your backup server as
well as the switches need to be prepared for this method to work smoothly. I assume
that you already have installed ``ssh`` and ``git``, user ``bkp3com`` is
created and it has ssh keys generated. The remaining steps are following:

#. Put the generated public key to a tftp server, for example::

      # cp /home/bkp3com/.ssh/id_rsa.pub /srv/tftp/bkp3com_rsa.pub

#. Prepare your switch with the following commands (``my-server`` is the server
   where the public key is available, should be IP or FQDN)::

      tftp my-server get bkp3com_rsa.pub

      system-view
        public-key local create rsa
        NO

        public-key local create dsa
        NO

        public-key peer bkp3com import sshkey bkp3com_rsa.pub
        NO

        local-user bkp3com
          service-type ssh level 3
        quit

        ssh user bkp3com service-type sftp
        ssh user bkp3com authentication-type publickey
        ssh user bkp3com assign publickey bkp3com
        sftp server enable

      quit
      delete bkp3com_rsa.pub
      YES
      save
      YES

      quit
    
   The above commands are prepared to be copy-pasted to your switch shell (so
   they contain also ``YES/NO`` responses to some questions that switch may
   optionally ask, and if it does not ask, these responses are treated as
   errors, but they are harmless). Don't forget to replace ``my-server``
   with your server name before pasting the code to the terminal.

#. Add the public key of the new switch to the ``.ssh/known_hosts`` of
   ``bkp3com`` user. The most straightforward method is to just connect
   to your switch via sftp (``switch-01`` is the IP or DNS name of your switch)::

      # su - bkp3com;
      # echo "quit" | sftp switch-01

   Answer ``yes`` to the question posed by ``sftp``.

LICENSE
-------

Copyright (c) 2014 by Pawel Tomulik <ptomulik@meil.pw.edu.pl>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE

.. <!--- vim: set expandtab tabstop=2 shiftwidth=2 syntax=rst: -->
