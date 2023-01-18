# Data Hugo Module

This module help starting to use the UX proposed for the final implementation of [Hugo building Pages from Data](https://github.com/gohugoio/hugo/issues/6310) in your current project. 

You should be aware of the current workaround to build Pages from Data with Hugo as seen [here](https://www.thenewdynamic.com/article/toward-using-a-headless-cms-with-hugo-part-2-building-from-remote-api/). This uses a second hugo project to build markdown files which are then mounted into the content directory of the main hugo project.

The module will read any `_content.{json|toml|yaml|hugo}` file in the main project's content directory to generate mardkown files in the prebuild project matching the structure. See [Usage](#usage)

## Requirements

Requirements:
- Go 1.14
- Hugo 0.103.0


## Installation

IMPORTANT! 
This module should be used on the ["prebuild"](https://www.thenewdynamic.com/article/toward-using-a-headless-cms-with-hugo-part-2-building-from-remote-api/#step-1) hugo project, and not the main project.

For this documentation, we'll presume your prebuild project lives under the `_prebuild` directory.

### 1. Create the prebuild project

Add a `_prebuild/config.yaml` to your project. 

### 2. [init](https://gohugo.io/hugo-modules/use-modules/#initialize-a-new-module) your prebuild project as Hugo Module. 

Move down to the `_prebuild` directory and:

```
$: hugo mod init {repo_url}/_prebuild
```

### 3. Edit your prebuild project's config file:

```yaml
# _prebuild/config.yaml
module:
  imports:
  - path: github.com/theNewDynamic/hugo-module-tnd-data
```

### 4. Mount settings (important)

In order for the module from the prebuild project to surface your main project's content's data file, you need to setup the following `mounts` config on the prebuild project:

```yaml
# _prebuild/config.yaml
module:
  imports:
  - path: github.com/theNewDynamic/hugo-module-tnd-data
  mounts:
  - source: ../content
    target: assets
  # Optional: If you want to use dynamically generated data with a `_content.hugo` file, 
  # you'll also need to mount the content dir in this project's partials.
  - source: ../content
    target: layouts/partials
    excludeFiles: 
    - "**.md"
  # Finally, if your `_content.hugo` files rely on some of the main project's partial, 
  # you'll need this final mount settings.
  - source: ../layouts/partials
    target: layouts/partials
```

### That's it.

Yes, your _prebuild project does not need any layouts files on its own.

## Usage

Each section or the root of your content directory can use it's own `_content` file. A data file can either `json`, `toml`, `yaml` or `hugo`/`html`. This could be the content directory of your projets:

```
content
├── blog
│   ├── _content.json
│   └── an-extra-local-post.md
├── travels
│   └── _content.yaml
├── products
│   └── _content.hugo
```

Each _content file should contain a Hugo readable version of an array of entries containing the following:

__basename__: this will be used to generate the file name. We'll use the directory and append this with `.md` So from `books/data.yaml` the file will be printed at `books/{basename}.md`
__filepath__: Can be used instead of the `basename` key to set a full filepath.
__data__: Will be printed as front matter.
__body__: (optinal) Will be printed as the file body (below front matter)

### Command

To generate the mardown files form the main project's directory:
```
hugo -s _prebuild
```
☝️ Where "_prebuild" is the directory of your prebuild Hugo project.


#### Examples

##### Harcoded ready to use Data

Here is an example of a very basic `books/_content.yaml` file containing an array of books.

_`/books/data.json`_:
```json
[
  {
    "basename": "my-first-book",
    "data": {
      "title": "My firt Book"
    }
  },
  {
    "filepath": "/not-book-dir/somewhere-else.md",
    "data": {
      "title": "My second book"
    },
    "body": "Donec id elit non mi porta gravida at eget metus. Aenean eu leo quam. \nPellentesque ornare sem [lacinia](/here) quam venenatis vestibulum. Donec sed odio dui. \nCras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis."
  }
]
```


So with the yaml file example above, the module will print two files in its books directory:

/public/books/my-first-book.md:

```
{"title":"My firt Book"}
```

/public/not-book-dir/somewhere-else.md:

```
{"title":"My second book"}

Donec id elit non mi porta gravida at eget metus. Aenean eu leo quam. 
Pellentesque ornare sem [lacinia](/here) quam venenatis vestibulum. Donec sed odio dui. 
Cras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis.
```

##### Dynamic Data

The above is fine for ready to use data. But if you need to format a tier produced data with the appropriate keys expected by the module, or use the powerful `resources.GetRemote` to fetch the data from an API, you'll need more power.

You can use a `_content.hugo` file which will, like a returning partial, return an array. The module will process it in order to generate markdown files.

```
{{/* /travels/_content.hugo */}}
{{ $travels := slice }}
{{ range site.Data.travels }}
  {{ $travels = $travels | append (dict
    "basename" .slug_string
    "data" .metadata
    "body" .contentMD 
  ) }}
{{ end }}
{{ return $travels}}
```


```
{{/* /monsters/_content.hugo */}}
{{ $monsters := slice }}
{{ with resources.GetRemote "https://monsters-api.netlify.app/?v=2" }} 
  {{ with .Content | unmarshal }}
    {{ range . }}
      {{ $monster := dict
        "basename" .id
        "data" (dict
          "title" .title
          "img" .img
        )
        "body" .content 
      }}
      {{ $monsters = $monsters | append $monster }}
    {{ end }}
  {{ end }}
{{ end }}
{{ return $monsters }}
```

### Settings

Settings are added to the project's parameter under the `tnd_data` map as shown below. Here are an example of the available keys set with their defaults.

```yaml
# config.yaml
params:
  tnd_data:
    data_file_basenames: _content
    fm_format: json
    quiet: false
```

#### Data File basename (data_file_basename)

Defaults to `_content`. If the file cannot be called `_content.*` for a given project, set the basename the module should look for.

#### Front Matter Format (fm_format)
The markdown files generated by the module are meant to be fed to Hugo, so readability should'nt be a problem. But for debugging purposes it might be useful to generate more "readable" markdown files. This allow the user to use `yaml` or `toml` for the generated files' Front Matter.

#### Quiet (quiet)

Defaults to `false`. By default the module will print a detailed list of all the files generated by a given `_content` file. Setting `quiet` to true will limit to log to each `_content` files treated.

```
From /posts/_content.json: 1,000 files
- posts/ante-and-11-at-nulla.md
- posts/nisl-and-49-at-in.md
- posts/justo-and-47-at-nec.md
- posts/nisl-and-48-at-sapien.md
- posts/quam-and-41-at-felis.md
- ...
```

### Data files and Syntax Highlighting

We understand that your IDE might not really know what to do with files ending in `.hugo`.
For this reason the module will also read data files ending with the `html` extension.

But as Hugo will choke on Go Template syntax inside its content directory, you will need to make use of the `ignoreFiles` settings to exclude you HTML data files from the main project content directory.

## theNewDynamic

This project is maintained and loved by [thenewDynamic](https://www.thenewdynamic.com).
