<center><img src="https://developers.google.com/open-source/gsoc/resources/downloads/GSoC-logo-horizontal.svg" alt="gsoc" height="50"/>
</center>
<center>
<img src="https://summerofcode.withgoogle.com/media/org/fortran-lang/ay9se7mc6vgdwgbn-360.png" height="45"/>
</center>

# Google Summer of Code 2022 Final Work Product

* **Name** : Arteev Raina
* **Organization** : Fortran-lang
* **Project** : fpm-ize the Fortran ecosystem
* **Mentors** : [Sebastian Ehlert](https://github.com/awvwgk), [Milan Curcic](https://github.com/milancurcic)
* **Discourse Profile** : [Profile](https://fortran-lang.discourse.group/u/arteevraina/summary)

## Abstract

Fortran Package Manager is  a build system and a package manager for Fortran projects.

 My project motive was to fpm-ize the Fortran Ecosystem by making the packages that are available in the Fortran Ecosystem fpm compatible. So, we had two ways to achieve this objective one was to work on the packages directly and change the code there but on the other hand we had another way to implement new features inside fpm only and lower the bar for the packages that use preprocessing extensively to get fpm compatible. So, I took up the problem to implement preprocessing support in Fortran Package Manager during the first week of community bonding itself so that I have enough time in understanding the project and its requirements. 

## Objectives Completed

### Added support for defining preprocessor table inside ```manifest``` file.

This was the first task that I took up while implementing preprocessing support inside fpm. The completion of this feature enabled the fpm users to add a table in their manifest file and define the preprocessor configuration. 

```toml
[preprocess.cpp]
suffixes = ["F90", "f90"]
directories = ["src/feature1", "src/models"]
macros = ["FOO", "BAR"]
```

The ```preprocess_config_t``` that is the type for preprocessor table defined in ```fpm.toml``` has the following member variables.

```Fortran
   !> Configuration meta data for a preprocessor
   type :: preprocess_config_t

      !> Name of the preprocessor
      character(len=:), allocatable :: name

      !> Suffixes of the files to be preprocessed
      type(string_t), allocatable :: suffixes(:)

      !> Directories to search for files to be preprocessed
      type(string_t), allocatable :: directories(:)

      !> Macros to be defined for the preprocessor
      type(string_t), allocatable :: macros(:)
```

### Added ablility inside fpm to add C Preprocessing compiler flag

In this task, I implemented the feature in fpm to add C Preprocessing flag if cpp preprocessor table is defined in ``fpm.toml`` file. The necessary condition inside the subroutine ```set_preprocessor_flags``` which handles this is as follows :

```Fortran
do i = 1, size(package%preprocess)
    if (package%preprocess(i)%name == "cpp") then
        flags = flag_cpp_preprocessor// flags
        exit
    end if
end do
```

In this way, fpm user can avoid passing the cpp preprocessor flag using command line like this
```fpm build --flag "-cpp -DFOO"```

### Add ability to read and parse the macros defined in the ```fpm.toml``` file.

In the first task, as I had already added the ability to define the macros inside the preprocessor table. So, now in this task I added the feature in fpm to parse and read those defined macros in the manifest and pass them as flags in the command for compilation of the files.

We can also define the valued macros in the manifest and also reuse value like Version number from the manifest as a value for a particular macro.

```toml
version = "1"

[preprocess]
[preprocess.cpp]
macros=["FOO", "-BAR=2", "VERSION={version}"]
```

While implementing this feature, I along with my mentors identified a specific problem with the implementation of macros parsing. The problem was that macro definition were made global and that would have created undesirable results in large projects as these macro definitions were passing to the dependencies of the current project as well. Therefore, we decided to keep these macros definitions local to the package type.

```Fortran
!> Type for describing a single package
type package_t

    !> Name of package
    character(:), allocatable :: name

    !> Array of sources
    type(srcfile_t), allocatable :: sources(:)

    !> List of macros.
    type(string_t), allocatable :: macros(:)

    !> Package version number.
    character(:), allocatable :: version

end type package_t
```

### Implement comipler wrapper ```fypp``` preprocessing script that compiles ```stdlib``` project using fpm.

My next objective that I decided with my mentors was to add ```fypp``` support inside fpm as it would allow us to compile ```stdlib``` using fpm.

```Fypp``` is a Python powered Fortran Preprocessor that stdlib uses and as of now fpm does not support ``fypp`` so we could not compile ``stdlib`` using ``fpm``.

In this script, we added fypp preprocessing on demand. Fpm can invoke this script by passing the path to the script file in the ``--compiler`` flag.

```Bash
fpm build --compiler "$PWD/fypp-gfortran.py"
```

We faced multiple problems while using this script to compile ``stdlib``. These include:

* Macros defined in the manifest were not available while compiling tests.

This was a quick fix as I was not adding macros while fpm was building target list specifically for the type ``FPM_UNIT_PROGRAM``.

```Fortran
case (FPM_UNIT_PROGRAM)

  call add_target(targets,package=model%packages(j)%name,type = FPM_TARGET_OBJECT,&
        output_name = get_object_name(sources(i)), &
        source = sources(i), &
        macros = model%packages(j)%macros, &
        version = model%packages(j)%version)
```

* Tests in the stdlib used functions that were defined in C++ files and fpm could not compile C++ files.
* We were not skipping some tests defined in ``test_mean_f03.fypp`` if the supported MAXRANK was not provided. 

### Add support for compiling C++ files inside fpm.

While compiling ``stdlib`` tests using ```fpm``` wrapper script I discovered that some of the functions referenced in the test files were defined in the C++ files and fpm is not able to compile the C++ files and properly link them. So, I worked on adding the C++ compilation support and with proper linking we were able to compile the tests which uses C++ files inside the ``stdlib``.

### Modify the structure of ```stdlib```

In order to compile the stdlib we had to modify its structure a bit in order to get the files in source directory, the tests and the example directory directly compiled using fpm with the help of compiler wrapper script using ```fypp```.

These changes include :
* Adding project specific ```fpm.toml``` file.

```toml

name = "stdlib"

[dev-dependencies]
test-drive.git = "https://github.com/fortran-lang/test-drive.git"

[build]
auto-examples = false
auto-tests = true

[preprocess.cpp]
macros = [
  "PROJECT_VERSION_MAJOR=0",
  "PROJECT_VERSION_MINOR=1",
  "PROJECT_VERSION_PATCH=0",
  "MAXRANK=4",
]
```

* Renaming submodules with same names & moving the ```common.fypp``` file inside include directory.

* Skipping the tests inside ``test_mean_f03.fypp`` when ```MAXRANK``` was not greater than 7.

## Objectives Left 

* The compiler wrapper script inside ``fypp-gfortran.py`` is still a temporary solution for adding ```fypp``` support in the fpm. In long term, we have planned to inject this script inside ``fpm`` and ```fpm``` should be able to handle the ``fypp`` preprocessing on its own without any external wrapper script.

* Hashing of included files to detect changes.

* Support for C++ modules.

## Pull Requests

| Pull Request | Description |
|----|----|
| [Added Basic Preprocess Table Configuration #715](https://github.com/fortran-lang/fpm/pull/715) | <ul><li>Adds a preprocess table configuration.</li><li>Add a compiler flag when ``[preprocess.cpp]`` is found in ``manifest`` file.</li></ul>|
| [Ability to parse and read macros from manifest #720](https://github.com/fortran-lang/fpm/pull/720) | <ul><li>Ability to read macros from the manifest.</li><li>Ability to reuse values like version number from manifest as a value for a macro.</li><li>Fix to keep  the macros package local.</li></ul> |
| [Added Compiler Wrapper script for Preprocessing using fypp #729](https://github.com/fortran-lang/fpm/pull/729) | <ul><li>Adds a compiler wrapper script for preprocessing the ``fypp`` files and storing them inside the build directory.</li><li>Makes the macros defined in manifest available while running tests.</li></ul> |
| [Added support for C++ files compilation #733](https://github.com/fortran-lang/fpm/pull/733) | <ul><li>Adds support for C++ files compilation.</li></ul> |
| [Structural Change in Stdlib #675](https://github.com/fortran-lang/stdlib/pull/675) | <ul><li>Adds a manifest file.</li><li>Rename sub modules with same name.</li><li>Skip Tests when ``MAXRANK`` is not supported.</li><li>Fix failing CI.</li></ul> |
| [Added Documentation for Preprocess Table #60](https://github.com/fortran-lang/fpm-docs/commit/fb0117e9890cceb96c0ad669e03f3591e57d3c8e) | <ul><li>Added documentation for ``preprocess`` table that can be defined in ``fpm.toml`` file.</li></ul> |
| [Updated Example Readme #740](https://github.com/fortran-lang/fpm/pull/740) | <ul><li>Updated the [``README.md``](https://github.com/fortran-lang/fpm/blob/main/example_packages/README.md) of example packages.</li></ul> |
| [Remove unnecessary space in fpm new cmd #684](https://github.com/fortran-lang/fpm/pull/684) | <ul><li>This Pull Request removed leading space in fpm new command.</li></ul>
| [Remove extra space from help-test cmd #686](https://github.com/fortran-lang/fpm/pull/686) | <ul><li>This is a similar fix as [#684](https://github.com/fortran-lang/fpm/pull/684) but for help-test.</li></ul>

## Weekly Blogs

| Week | Blog Links |
|----|----|
| Week 1 <br> (23-05-2022) | [# Blog 1](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-1/3521)|
| Week 2 <br> (30-05-2022) | [# Blog 2](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-2/3601) |
| Week 3 <br> (06-06-2022) | [# Blog 3](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-3/3669) |
| Week 4 <br> (20-06-2022) | [# Blog 4](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-4/3803) |
| Week 5 <br> (27-06-2022) | [# Blog 5](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-5/3843) |
| Week 6 <br> (04-07-2022) | [# Blog 6](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-6/3911) |
| Week 7 <br> (11-07-2022) | [# Blog 7](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-7/3975) |
| Week 8 <br> (18-07-2022) | [# Blog 8](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-8/4020) |
| Week 9 <br> (25-07-2022) | [# Blog 9](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-9/4069) |
| Week 10 <br> (01-08-2022) | [# Blog 10](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-10/4107) |
| Week 11 <br> (08-08-2022) | [# Blog 11](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-11/4133) |
| Week 12 <br> (15-08-2022) | [# Blog 12](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-12/4171) |
| Week 13 <br> (22-08-2022) | [# Blog 13](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-13/4198) |
| Week 14 <br> (29-08-2022) | [# Blog 14](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-14/4229) |
| Week 15 <br> (05-09-2022) | [# Blog 15](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-15/4266) |
| Week 16 <br> (11-09-2022) | [#Blog 16](https://fortran-lang.discourse.group/t/gsoc-22-fpm-ize-the-fortran-ecosystem-blog-post-by-arteev-raina-16/4295)

#### Link to the [Weekly Meeting Notes](https://fortran-lang.discourse.group/t/gsoc-meeting-notes-fpm-ize-the-fortran-ecosystem/3641).

## Acknowledgement
I would like to thank my mentors [Sebastian Ehlert](https://github.com/awvwgk), [Milan Curcic](https://github.com/milancurcic), [Laurence Kedward](https://github.com/LKedward), [Jeremie Vandenplas](https://github.com/jvdp1), [Zoziha](https://github.com/zoziha) and other community members as well because of their support I was able to progress and complete my objectives. They also helped me in understanding some crucial parts of Fortran code whenever I felt stuck while solving issues. They always responsded to my queries whether it was related to code or anything related to program. While contributing to this project, I learned a lot about writing Fortran, writing maintainable and clean code. I look forward to keep contributing to the project even after the completion of GSoC program because of great learning experience that I have had while contributing to this organization and will continue to have.