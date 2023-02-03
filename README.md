# Data Hugo Module

## Build Pages from Data now!

This module uses the UX proposed for the final implementation of [Hugo building Pages from Data](https://github.com/gohugoio/hugo/issues/6310) while relying on a [workaround](https://www.thenewdynamic.com/article/toward-using-a-headless-cms-with-hugo-part-2-building-from-remote-api/) to achieve the same result.

This workaround uses a second hugo project to build markdown files which are then mounted into the content directory of the main hugo project.

The module will read any `_content.{json|toml|yaml|hugo}` file in the main project's content directory to generate mardkown files in the prebuild project matching the structure. See [Usage](#usage)

## Requirements

Requirements:
- Go 1.14
- Hugo 0.103.0


## Installation

IMPORTANT! 
This module should be used on the ["prebuild"](https://www.thenewdynamic.com/article/toward-using-a-headless-cms-with-hugo-part-2-building-from-remote-api/#step-1) hugo project, and not the main project.

For this documentation, we'll assume your prebuild project lives under the `_prebuild` directory.

### 1. Create the prebuild project

Add a `_prebuild/config.yaml` to your main Hugo project. 

### 2. [init](https://gohugo.io/hugo-modules/use-modules/#initialize-a-new-module) your prebuild project as Hugo Module. 

Move down to the `_prebuild` directory and:

```
$: hugo mod init {repo_url}/_prebuild
```

Where `{repo_url}` is your main project's repo url ex: `hugo mod init github.com/mememe/my-website/_prebuild`

### 3. Edit your prebuild project's config file:

```yaml
# _prebuild/config.yaml
module:
  imports:
  - path: github.com/theNewDynamic/hugo-module-tnd-data
```

### 4. Mount settings for _prebuild

In order for the module from the prebuild project to surface your main project's content's "data" files, you need to setup the following `mounts` config on the prebuild project:

```yaml
# _prebuild/config.yaml
module:
  imports:
  - path: github.com/theNewDynamic/hugo-module-tnd-data
  mounts:
  - source: ../content
    target: assets
  # Optional: If you want to use dynamically generated data with a `_content.hugo` file, 
  # you'll also need to mount the content dir in this project's partials (but not the md files).
  - source: ../content
    target: layouts/partials
    excludeFiles: 
    - "**.md"
  # Finally, if your `_content.hugo` files rely on some of the main project's partial, 
  # you'll need this final mount settings.
  - source: ../layouts/partials
    target: layouts/partials
```

### 5. Mount settings for the main project.

Now we need to ensure the main project will be using the markdown files generated by the prebuild project:

```yaml
# /config.yaml
module:
  mounts:
  - source: _prebuild/public
    target: content
```

If used in a multilingual project relying on directories, be aware of the `lang` key as illustrated [here](https://www.regisphilibert.com/blog/2018/08/hugo-multilingual-part-1-managing-content-translation/#with-hugo-module-mounts).

### That's it?

Yes, your prebuild project does not need any layouts files on its own, it'll be using the module's.

## Usage
 
Each section or the root of your main project's content directory can use it's own `_content` file. A data file can either be `json`, `toml`, `yaml` or `hugo`/`html`. This could be the content directory of your project:

```
content
├── _content.yaml
├── blog
│   ├── _content.json
│   └── an-extra-local-post.md
├── travels
│   └── _content.yaml
├── products
│   └── _content.hugo
```

Each _content file should contain a Hugo readable version of an array of entries where each entry is an object containing the following:

__destination*__: A string. Can either be in the form of a slug (ex: `my-web-page`) in which case the filepath will be deducted or a full filepath with extension `/books/city-on-a-hill.md`.

__data__: An object. This will be printed as the Front Matter of the entry.

__body__: A string. This will be printed as the entry's body (below front matter)

### Command

To generate the mardown files form the main project's directory:
```
hugo -s _prebuild
```
☝️ Where "_prebuild" is the directory of your prebuild Hugo project.

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

Defaults to `_content`. 
If the file cannot be called `_content.*` for a given project, use this setting to set the basename the module should look for.

#### Front Matter Format (fm_format)
Defaults to `json`. 
Available formats are `json`, `yaml`, `toml`.
The markdown files generated by the module are meant to be fed to Hugo, so the poor readability of minified JSON should'nt be a problem. But for debugging purposes it might be useful to generate more "readable" markdown files. This allow the user to use `yaml` or `toml` for the generated files' Front Matter. 

#### Quiet (quiet)

Defaults to `false`. 
By default the module will print a detailed list of all the files generated by a given `_content` file. Setting `quiet` to true will limit to log to each `_content` files treated.

```
From /posts/_content.json: 1,000 files
- posts/ante-and-11-at-nulla.md
- posts/nisl-and-49-at-in.md
- posts/justo-and-47-at-nec.md
- posts/nisl-and-48-at-sapien.md
- posts/quam-and-41-at-felis.md
- ...
```

## Examples

### Harcoded, ready to use data.

Here is an example of a very basic `books/_content.json` file containing an array of books.

_`/books/data.json`_:
```json
[
  {
    "destination": "my-first-book",
    "data": {
      "title": "My firt Book"
    }
  },
  {
    "destination": "/not-book-dir/somewhere-else.md",
    "data": {
      "title": "My second book"
    },
    "body": "Donec id elit non mi porta gravida at eget metus. Aenean eu leo quam. \nPellentesque ornare sem [lacinia](/here) quam venenatis vestibulum. Donec sed odio dui. \nCras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis."
  }
]
```


So with the file example above, the module will print two files in its public directory:

The first one at `/public/books/my-first-book.md`:

```
{"title":"My firt Book"}
```

The second one at `/public/not-book-dir/somewhere-else.md`:

```
{"title":"My second book"}

Donec id elit non mi porta gravida at eget metus. Aenean eu leo quam. 
Pellentesque ornare sem [lacinia](/here) quam venenatis vestibulum. Donec sed odio dui. 
Cras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis.
```

### Dynamic Data

The above is fine for ready to use data. But if you need to format a tier produced data with the appropriate keys expected by the module, or use the powerful `resources.GetRemote` to fetch the data from an API, you'll need more power.

You can use a `_content.hugo` file which will at as a returning partial and return an array. The module will then process it in order to generate markdown files.

#### from a local Data file.
```
{{/* /travels/_content.hugo */}}
{{ $travels := slice }}
{{ range site.Data.travels }}
  {{ $travel := dict
    "destination" .slug_string
    "data" .metadata
    "body" .contentMD 
  }}
  {{ $travels = $travels | append $travel }}
{{ end }}
{{ return $travels}}
```

#### from a remote API
```
{{/* /monsters/_content.hugo */}}
{{ $monsters := slice }}
{{ with resources.GetRemote "https://monsters-api.netlify.app/?v=2" }} 
  {{ with .Content | unmarshal }}
    {{ range . }}
      {{ $monster := dict
        "destination" .id
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

## Notes

### A note on the `destination` setting.

**If the extension is ommited**, the string will be treated as a slug and the destination is detucted using the latter and the directory where the `_content` file is located.

**If an extension is included**, the module will use the string as is and skip the deduction logic.

Ex: From a `blog/_content.yaml` file, the entry with `destination: "travels/trip-in-belize"` will be saved at `blog/travels/trip-in-belize.md` while another entry from the same file with `destination: "americas/trip-in-belize.md"` will be saved at `/americas/trip-in-belize.md`

### A note on generating Data files. 

The module is agnostic on the kind of file created. With a destination containing a `.json` extension and a `data` object your file can be published and mounted as a data file. Just make sure your `_content` file returns an array even if you only need one file!

For example from a `_content.hugo` file which could but &mdash; does not have to &mdash; live at the root of your main project's content directory.

```
{{ $data := dict }}
{{ with resource.GetRemote "/api/global/data/" }}
  {{ with .Content | unmarshal }}
    {{ $data = dict 
      "destination" "data/settings.json"
      "data" .
    }}
  {{ end }}
{{ end }}
{{ return slice $data }}
```

And the new mounting setting, still on the main project:

```yaml
# /config.yaml
module:
  mounts:
    [...]
    - source: _prebuild/public/data
      target: data
```

### A note on `_content` files and syntax highlighting

We understand that your IDE might not really know what to do with files ending in `.hugo`.
For this reason the module will also read `_content` files ending with the `.html` extension.

But as Hugo will choke on Go Template syntax inside its content directory, you will need to make use of the `ignoreFiles` settings to exclude you HTML data files from the main project's content directory.

```yaml
# /config.yaml
ignoreFiles:
- '\_content.html$'
```

### A note on `--cleanDestinationDir`

It is wise to use `--cleanDestinationDir` when building the prebuild project to ensure deleted entries are not kept in its public directory. But there is a current bug in Hugo wich ignores this flag if a `static` directory is not present in the project. You can solve this by simply adding a `_prebuild/static/.keep` file.

## theNewDynamic

This project is maintained and loved by [thenewDynamic](https://www.thenewdynamic.com).
