---
title: Split Large Files into Lines with NodeJS
description: Avoid memory issues and invalid string length errors
author: James Ingold
published: true
---

Splitting a file into lines is pretty straight forward when working with smaller files, however larger files can run into memory issues. The convenient way to split a file into lines with NodeJS is to use fs.readFile and then split it by the newline character. You can even make it a one liner:

```javascript
const lines = fs.readFileSync(filePath).toString("UTF8").split("\n");
```

We are using fs.readFileSync to read the contents of the file and then converting the buffer into a string before splitting it with the newline character. This method runs into issues when dealing with a large file though. Since we're reading the file into memory as a string before splitting lines this can cause a memory exception. NodeJS also has a character limit for the String type of 512 MB. Trying to save a string into memory greater than that limit will result in an invalid string length error.

##### Handling Large Files

A better option is to read the file into a buffer and then split the buffer into an array of buffers by line delimiter. This approach is more efficient for large files because it allows you to process the file in chunks without reading the entire file into memory at once.

Unfortunately, there is not a built-in way in NodeJS to split a buffer by a character. Here are a couple different options:

##### Option 1: [buffer-split package](https://www.npmjs.com/package/buffer-split)

```javascript
const bsplit = require("buffer-split");
const buf = fs.readFileSync(fileName);
const delim = Buffer.from("\n");
const result = bsplit(buf, delim);
const lines = _.map(result, (r) => {
  return r.toString("utf8");
});
```

The downside with this package is that it was published in 2016 and hasn't received any updates since. There are no vulnerabilities currently.

##### Option 2: [inter-ops package](https://github.com/vitaly-t/iter-ops)

```javascript
import { pipe, split } from "iter-ops";

const delim = Buffer.from("\n");
const i = pipe(
  buffer,
  split((value) => value === delim)
);
```

##### Option 3: Roll Your Own

```javascript
function splitBufferToArray(buf, splitStr) {
  const bufArr = [];
  let start = 0;
  let idx = buf.indexOf(splitStr, start);
  while (idx !== -1) {
    const chunk = buf.slice(start, idx);
    bufArr.push(chunk);
    start = idx + 1;
    idx = buf.indexOf(splitStr, start);
  }
  if (bufArr.length > 0) bufArr.push(buf.slice(idx));
  return bufArr;
}
```

The correct approach to dealing with large files is to read in the file and then split that buffer into chunks. This will allow you to process the file efficiently and avoid running into memory issues.

###### References:

[Maximum String Length NodeJS](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length)
