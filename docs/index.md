# Documentation intro

Welcome to the documentation for the Unreal Engine–based Virtual Anatomy application built by [JRCZ Data Science Lab](https://jrcz.nl/en/data-science-lab.html).
This documentation aims to help you navigate the codebase, understand the system’s architecture, and get up to speed with development and deployment processes.

## General overview

The project itself is divided into several repositories:

- Unreal Engine's Virtual Anatomy application [GitHub](https://github.com/jrcz-data-science-lab/Unreal-Engine-3D-Anatomy)
    - It is core of the whole project. This repository contains an Unreal Engine project that can be loaded into UE5.
- Python script for importing the Blender models to the Unreal Engine [GitHub](https://github.com/jrcz-data-science-lab/blender-scripts)
- Python script for generating JSON files that can be loaded in the Model tagging application [GitHub](https://github.com/jrcz-data-science-lab/Anatomy-blender-auto-prepare)     
- This documentation [GitHub](https://github.com/jrcz-data-science-lab/3D-Anatomy-Documentation)
- The first release [Onedrive](https://myhz-my.sharepoint.com/:f:/g/personal/li0047_hz_nl/IgDa2FBsKPF0SIZA0qtlAjwTAU-54GVRy-A9ipBUTyAYI3M?e=XKASmZ)

## How to approach this (large) project?

This project is large and can feel overwhelming to grasp everything at once. We all have tried to make as many components as stable and unchanging as possible. This means that most of what you see on the screen does not require a deep understanding right away—working code is best left untouched. 😄

If you are (completely or to some extent) not familiar with C++, we strongly encourage you to focus on learning this programming language, especially general OOP concepts and how pointers work, for the first 1–3 week(s) of your internship. Try building for example a simple calculator or a rock-paper-scissors app using OOP and pointers to get more comfortable with C++.

Next, dive into general Unreal Engine tutorials. Remember, everything in Unreal Engine is highly decoupled — it is not like a website or other applications where you can easily call everything through dependency injections or setters. We also recommend following some C++ tutorials in Unreal Engine and creating a small project to test things out. Once you get comfortable with the C++ side, using Blueprints will feel like a helpful tool for tying your logic together.

Keep in mind that C++, OOP, and other programming patterns are not quite the same when developing in Unreal Engine. Moreover, Unreal Engine documentation can be extremely frustrating to navigate. Because of this, we strongly advise using JetBrains Rider IDE if your RAM allows it, as it provides the best autocompletion and function documentation support. Otherwise, you can try increasing your swap file to use your SSD as temporary RAM.

---

!!! Note

    We have made an effort to cover as much as possible. However, as you will soon realize, balancing the development of new features, handling with the HZ internship flow/process, and keeping documentation up to date is no small feat.

    While this documentation may not be perfect, it should serve as a solid guide to help you get started and make sense of the project.
