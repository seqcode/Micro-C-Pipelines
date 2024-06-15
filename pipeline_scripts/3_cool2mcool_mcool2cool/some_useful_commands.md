1) Command to print resolutions in .mcool file <br>
`cooler ls <path-to-mcool-file>`

2) Command to list chromosomes in .cool file <br>
`cooler dump -t 'chroms' <path-to-cool-file>` <br> <br>
You can select particular .cool file from .mcool file using this format - <.mcool-file-name>::/resolutions/<resolution> <br>
E.g. `cooler dump -t 'chroms' "hPSC-lib545+lib546_merged.mcool::/resolutions/1000"`
