---
state: draft
---

# Installing nodes via tgz upload in editor

## Summary

The `/nodes` api provides a number of ways to specify a module to be installed:

 - by name
 - by path to folder containing the code
 - by url to an installable thing (which requires the name of the module to be provided as part of the api call as well)

This design note proposes allowing the user to install a node by uploading its tgz file via the editor.

## Authors

 - @knolleary

## Details

This feature will *only* be available if the default LocalFileSystemStorage plugin is
being used. This is because it requires having persistent file storage that cannot be
solved with the `autoInstallModules` option.

<details><summary>Why not `autoInstallModules`</summary>
When the `autoInstallModules` flag is set, the runtime will automatically `npm install`
any modules it knows it had previously (from `.config.json`) but fails to find on
startup. This assumes the modules can be installed based just on their module name.

Once we support uploading tgz files, without persistent file storage, the tgz file
may no longer be available locally when node-red restarts - so a reinstall will
not be possible.
</details>

### `/nodes` api updates

The `/nodes` end-point will now accept a POST request with a content-type of `multipart/form-data`.

It will accept a single file uploaded with the field name `tarball`. If this field
is set, none of the other fields (`name`,`version`,`url`) should be provided.

The following `curl` command would be a valid way to call the endpoint:

```
curl -v -F tarball=@/tmp/node-red-node-random-0.2.0.tgz http://localhost:1880/nodes
```

### Install behaviour

 - The `tgz` file will first be unpacked to a temporary directory so the runtime can
   examine its contents.
 - The `tgz` file will then be saved to `<userDir>/nodes/<filename.tgz>`, where
   `filename.tgz` is normalised to `name-of-module-x.y.z.tgz` (the standard format
   generated by `npm pack`). This normalisation step is needed because we cannot
   rely on the original `tgz` filename being accurate.
 - Node-RED will run `npm install <userDir>/nodes/<filename.tgz>`
   and then load the module as it does any other module.
 - If we detect this is an upgrade of an existing module that has a local `tgz`
   file, we will remove the old versions tgz file

The user's `package.json` file will be updated by npm with an entry:

```
   ...
   "module-name": "file:./nodes/<filename.tgz>"
   ...
```

By having the `tgz` files in the `<userDir>/nodes` directory they can be easily
backed up alongside the flow files and package.json. Reinstalling the modules can
be done by running `npm install` in `<userDir>`.

#### Projects

Some extra thought is needed about how projects can include the tgz files. They are
installed into the runtime, just like other modules, so projects will be able to
use the nodes. But should the tgz be added to the project's files so they get version
controlled? Without that, it would mean more manual work is needed to clone and
run the project elsewhere - a simple `npm install` wouldn't be sufficient.

The UI tries to help the user keep their project's package.json file up to date, but could do a much better job of it. Maybe the UI could allow the user to optionally
copy the tgz into their project's directory (still under a `nodes` directory).


### Palette Editor enhancements

The `install` tab of the palette editor will have an upload button added to the
toolbar. Clicking on the button will prompt the user to select a file to upload.

It will then show a progress bar as the file is being uploaded and then a similar
view to when clicking on install for a node in the catalog; a spinner and a button
to view the event log.

### Configuration options

The Palette Editor can already be disabled by setting `editorTheme.palette.editable`
to `false`.

As some embedded uses of Node-RED may want to have more control over what can be
installed into the runtime, the ability to upload a tgz can be disabled by setting
`editorTheme.palette.upload` to `false`.

### Runtime API changes

The `runtime.nodes.addModule` API will be updated to accept a `tarball` property
in its options argument.

This will be an object of the form:
```json
{
    "name": "<filename>",
    "size": "<size in bytes>",
    "buffer": "<tgz file contents in a Buffer>"
}
```

If this property is set, then none of `module`, `version` or `url` may also be set. If
any of them are, the call with reject with an error with the code `invalid_request` and
a status `400`.


## History

- 2020-08-12 - Initial proposal submitted
