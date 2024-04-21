Getting Backtraces in ROS 2
===========================

.. contents:: Table of Contents
   :depth: 2
   :local:

**Goal:**  Show various methods for getting backtraces in ROS 2 

**Tutorial level:** Intermediate

**Time:** 15 minutes

The following steps show ROS 2 users how to get traces when they encounter a problem.

Overview
========

This document explains one set of methods for getting backtraces for ROS 2.
There are many ways to accomplish this, but this is a good starting point for new C++ developers without GDB experience.

The following steps show ROS 2 users how to get traces from specific nodes when they encounter a problem.
This tutorial applies to both simulated and physical robots.

This will cover how to get a backtrace from a specific node using ``ros2 run``, from a launch file representing a single node using ``ros2 launch``, and from a more complex orchestration of nodes.
By the end of this tutorial, you should be able to get a backtrace when you notice a node crashing in ROS 2.

Preliminaries
=============

GDB is the most popular debugger for C/C++ on Unix systems.
It can be used to determine the reason for a crash and track threads.
It may also be used to add breakpoints in your code to check values in memory at particular points in your software.

Using GDB is a critical skill for all software developers working on C/C++.
Many IDEs will have some kind of debugger or profiler built in, but with ROS 2, there are few IDEs to choose.
Therefore it's important to understand how to use these raw tools you have available rather than relying on an IDE to provide them.
Further, understanding these tools is a fundamental skill of C/C++ development and leaving it up to your IDE can be problematic if you change roles and no longer have access to it or are doing development on the fly through an ssh session to a remote asset.

Using GDB luckily is fairly simple after you have the basics under your belt.
The first step is to add ``-g`` to your compiler flags for the ROS package you want to profile / debug.
This flag builds debug symbols that GDB can read to tell you specific lines of code in your project are failing and why.
If you do not set this flag, you can still get backtraces but it will not provide line numbers for failures.

Using ``--cmake-args -DCMAKE_BUILD_TYPE=Debug`` in your ``colcon build`` command will add the ``-g`` flag to your build.

.. code-block:: bash

  colcon build --packages-up-to <package_name> --cmake-args -DCMAKE_BUILD_TYPE=Debug 


Now you're ready to debug your code!
If this was a non-ROS project, at this point you might do something like below.
Here we're launching a GDB session and telling our program to immediately run.
Once your program crashes, it will return a gdb session prompt denoted by ``(gdb)``.
At this prompt you can access the information you're interested in.
However, since this is a ROS project with lots of node configurations and other things going on, this isn't a great option for beginners or those that don't like tons of commandline work and understanding the filesystem.

.. code-block:: bash

  gdb ex run --args /path/to/exe/program

Below are sections to describe the 3 major situations you could run into with ROS 2-based systems. 
Read the section that best describes the problem you're attempting to solve.

Debugging a specific node with GDB
==================================

To easily set up a GDB session before launching a ROS 2 node, leverage the ``--prefix`` option in launch files. 
This option allows you to specify a command to execute before the node starts. 
For GDB debugging, use it as follows:

.. note::

  Keep in mind that a ROS 2 executable might contain multiple nodes. 
  The ``--prefix`` approach ensures you're debugging the correct node within the process.

**Why Direct GDB Usage Can Be Tricky**

``--prefix`` will execute some bits of code before our ROS 2 command allowing us to insert some information. 
If you attempted to do ``gdb ex run --args ros2 run <pkg> <node>`` as analog to our example in the preliminaries, you’d find that it couldn’t find the ``ros2`` command. 
Additionally, trying to source your workspace within GDB would fail for similar reasons. 
This is because GDB, when launched this way, lacks the environment setup that normally makes the ``ros2`` command available.

**Simplifying the Process with --prefix**

Rather than having to revert to finding the install path of the executable and typing it all out, we can instead use ``--prefix``. 
This allows us to use the same ``ros2 run`` syntax you’re used to without having to worry about some of the GDB details.

.. code-block:: bash

  ros2 run --prefix 'gdb -ex run --args' <pkg> <node> --all-other-launch arguments 

**The GDB Experience**

Just as before, this prefix will launch a GDB session and run the node you requested with all the additional command-line arguments. 
You should now have your node running and should be chugging along with some debug printing.

Once your node crashes, you’ll see a prompt like below.
At this point you can get a backtrace.

.. code-block:: bash

  (gdb)

In this session, type ``backtrace`` and it will provide you with a backtrace.
Copy this for your needs.
For example:

.. code-block:: bash

  (gdb) backtrace
  #0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:50
  #1  0x00007ffff79cc859 in __GI_abort () at abort.c:79
  #2  0x00007ffff7c52951 in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
  #3  0x00007ffff7c5e47c in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
  #4  0x00007ffff7c5e4e7 in std::terminate() () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
  #5  0x00007ffff7c5e799 in __cxa_throw () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
  #6  0x00007ffff7c553eb in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
  #7  0x000055555555936c in std::vector<int, std::allocator<int> >::_M_range_check (
      this=0x5555555cfdb0, __n=100) at /usr/include/c++/9/bits/stl_vector.h:1070
  #8  0x0000555555558e1d in std::vector<int, std::allocator<int> >::at (this=0x5555555cfdb0, 
      __n=100) at /usr/include/c++/9/bits/stl_vector.h:1091
  #9  0x000055555555828b in GDBTester::VectorCrash (this=0x5555555cfb40)
      at /home/steve/Documents/nav2_ws/src/gdb_test_pkg/src/gdb_test_node.cpp:44
  #10 0x0000555555559cfc in main (argc=1, argv=0x7fffffffc108)
      at /home/steve/Documents/nav2_ws/src/gdb_test_pkg/src/main.cpp:25

In this example you should read this in the following way, starting at the bottom:

- In the main function, on line 25 we call a function VectorCrash.

- In VectorCrash, on line 44, we crashed in the Vector's ``at()`` method with input ``100``.

- It crashed in ``at()`` on STL vector line 1091 after throwing an exception from a range check failure.

These traces take some time to get used to reading, but in general, start at the bottom and follow it up the stack until you see the line it crashed on.
Then you can deduce why it crashed.
When you are done with GDB, type ``quit`` and it will exit the session and kill any processes still up.
It may ask you if you want to kill some threads at the end, say yes.

From a Launch File
==================

Just as in our non-ROS example, we need to setup a GDB session before launching our ROS 2 launch file.
While we could set this up through the commandline, we can instead make use of the same mechanics that we did in the ``ros2 run`` node example, now using a launch file.

In your launch file, find the node that you’re interested in debugging.
For this section, we assume that your launch file contains only a single node (and potentially other information as well). 
The ``Node`` function used in the ``launch_ros`` package will take in a field prefix taking a list of prefix arguments. 
We will insert the GDB snippet here. 
**Consider the following approaches, depending on your setup:**

- **Local Debugging with Windowing System:**  If you are debugging locally and have a windowing system available, use:

.. code-block:: bash

  prefix=['xterm -e gdb -ex run --args']

This will provide a more interactive debbuging experience.
Example usecase for debugging building upon ``'start_sync_slam_toolbox_node'`` - 

.. code-block:: python 

  start_sync_slam_toolbox_node = Node(
    parameters=[
        get_package_share_directory("slam_toolbox") + '/config/mapper_params_online_sync.yaml',
        {'use_sim_time': use_sim_time}
    ],
    package='slam_toolbox',
    executable='sync_slam_toolbox_node',
    name='slam_toolbox',
    prefix=['xterm -e gdb -ex run --args'],  # For interactive GDB in a separate window
    output='screen')

- **Remote Debugging (No Windowing System):** If debugging remotely without a windowing system, omit ``xterm -e`` :

.. code-block:: bash

  prefix=['gdb -ex run --args']

GDB's output and interaction will happen within the terminal session where you launched the ROS 2 application.
Here's an similar example for the ``'start_sync_slam_toolbox_node'`` -

.. code-block:: python

  start_sync_slam_toolbox_node = Node(
    parameters=[
        get_package_share_directory("slam_toolbox") + '/config/mapper_params_online_sync.yaml',
        {'use_sim_time': use_sim_time}
    ],
    package='slam_toolbox',
    executable='sync_slam_toolbox_node',
    name='slam_toolbox',
    prefix=['gdb -ex run --args'],  # For GDB within the launch terminal
    output='screen')

Just as before, this prefix will launch a GDB session, now in ``xterm`` and run the launch file you requested with all the additional launch arguments defined.

Once your server crashes, you'll see a prompt like below, now in the ``xterm`` session. 
At this point you can now get a backtrace.

.. code-block:: bash

  (gdb)

In this session, type ``backtrace`` and it will provide you with a backtrace.
Copy this for your needs.
See the example trace in the section above for an example.

These traces take some time to get used to reading, but in general, start at the bottom and follow it up the stack until you see the line it crashed on.
Then you can deduce why it crashed.
When you are done with GDB, type ``quit`` and it will exit the session and kill any processes still up.
It may ask you if you want to kill some threads at the end, say yes.

From a Large Project
====================

Working with launch files with multiple nodes is a little different so you can interact with your GDB session without being bogged down by other logging in the same terminal.
For this reason, when working with larger launch files, its good to pull out the specific server you're interested in and launching it seperately.
These instructions are targeting ROS 2, but are applicable to any large project with many nodes of any type in a series of launch file(s).

As such, for this case, when you see a crash you'd like to investigate, its beneficial to separate this server from the others.

If your server of interest is being launched from a nested launch file (e.g. an included launch file) you may want to do the following:

- Comment out the launch file inclusion from the parent launch file

- Recompile the package of interest with ``-g`` flag for debug symbols

- Launch the parent launch file in a terminal

- Launch the server's launch file in another terminal following the instructions in `From a Launch File`_.

Alternatively, if your node of interest is being launched in these files directly (e.g. you see a ``Node``, ``LifecycleNode``, or inside a ``ComponentContainer``), you will need to seperate this from the others:

- Comment out the node's inclusion from the parent launch file

- Recompile the package of interest with ``-g`` flag for debug symbols

- Launch the parent launch file in a terminal

- Launch the server's node in another terminal following the instructions in `From a Node`_.

.. note::

  In this case you may need to remap or provide parameter files to this node if it was previously provided by the launch file.
  Using ``--ros-args`` you can give it the path to the new parameters file, remaps, or names.
  See :doc:`this tutorial <../../Guides/Node-arguments.html>` for the commandline arguments required.

  We understand this can be a pain, so it might encourage you to rather have each node possible as a separately included launch file to make debugging easier. 
  An example set of arguments might be ``--ros-args -r __node:=<node_name> --params-file /absolute/path/to/params.yaml`` (as a template).

Once your node crashes, you'll see a prompt like below in the specific server's terminal. At this point you can now get a backtrace.

.. code-block:: bash

  (gdb)

In this session, type ``backtrace`` and it will provide you with a backtrace.
Copy this for your needs.
See the example trace in the section above for an example.

These traces take some time to get used to reading, but in general, start at the bottom and follow it up the stack until you see the line it crashed on.
Then you can deduce why it crashed.
When you are done with GDB, type ``quit`` and it will exit the session and kill any processes still up.
It may ask you if you want to kill some threads at the end, say yes.

Debugging tests with GDB
========================

If a C++ test is failing, GDB can be used directly on the test executable in the build directory.
Ensure to build the code in debug mode.
Since the previous build type may be cached by CMake, clean the cache and rebuild.

.. code-block:: console

  colcon build --cmake-clean-cache --mixin debug

In order for GDB to load debug symbols for any shared libraries called, make sure to source your environment.
This configures the value of ``LD_LIBRARY_PATH``.

.. code-block:: console

  source install/setup.bash

Finally, run the test directly through GDB.
For example:

.. code-block:: console

  gdb -ex run ./build/rcl/test/test_logging

If the code is throwing an unhandled exception, you can catch it in GDB before gtest handles it.

.. code-block:: console

  gdb ./build/rcl/test/test_logging
  catch throw
  run

Automatic backtrace on crash
============================

The `backward-cpp <https://github.com/bombela/backward-cpp>`_ library provides beautiful stack traces, and the `backward_ros <https://github.com/pal-robotics/backward_ros/tree/foxy-devel>`_ wrapper simplifies its integration.

Just add it as a dependency and `find_package` it in your CMakeLists and the backward libraries will be injected in all your executables and libraries.
