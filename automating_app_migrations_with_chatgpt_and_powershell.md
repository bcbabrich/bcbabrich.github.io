# (Partially) Automating App Migrations Using ChatGPT and PowerShell

## Death by 1,000 tabs

My company is in the process of migration their on-premises datacenter to the cloud. With each app that we migrate comes a whole slew of potential issues, and so we need several different browser services to debug, such as the source code for the app, our app build platform, app deployment platform, the app itself, app-monitoring tools... and the list goes on.

For a single app like this one
```
https://github.com/my-organization/prefix_my-stellar-microservice
```

I might need to visit these services to debug:
```
"https://github.com/my-organization/prefix_my-stellar-microservice",
"https://github.com/my-organization/prefix_my-stellar-microservice-config/foo.yaml"
"https://deployment.platform/my-namespace?name=my-stellar-microservice"
"https://build.platform/#/applications/my-stellar-microservice"
"https://monitoring.platform/{possibly_very_long_filter}params:(query:my-stellar-microserviceBase){more_long_filter}
"https://monitoring.platform/{filter_for_different_type_of_log}params:(query:my-stellar-microserviceBase){more_long_filter}
```

I found that I kept losing focus while opening up each service, logging in, searching for an app name, copying the app name over to a second service, and all the little steps in between. So I began to look for a way to generalize this process.

I am going to show how I went from a manual process of navigating these apps to having a PowerShell script which jumps to each one in a short amount of time using ChatGPT.

## Abstracting and Obfuscating

The reader by now has noticed that the app I used in my example is not a real app. When we interact with LLMs, we have to be careful not to leak any sensitive information out to them by copying and pasting directly. We need to identify which parts of our input need to be parameterized and therefore hidden from the LLM. This also helps us abstract whatever process we're trying to capture.

Once we identify the parts of the URLs which do not change we can do a find and replace with a variable instead. In this case, we use `$Prefix` and `$Microservice`

```
"https://github.com/my-organization/$Prefix_$Microservice",
"https://github.com/my-organization/$Prefix_$Microservice-config/foo.yaml"
"https://deployment.platform/my-namespace?name=$Microservice"
"https://build.platform/#/applications/$Microservice"
"https://monitoring.platform/{possibly_very_long_filter}params:(query:$MicroserviceBase){more_long_filter}
"https://monitoring.platform/{filter_for_different_type_of_log}params:(query:$MicroserviceBase){more_long_filter}"
```

This lets us ask ChatGPT to write a PowerShell script for us that opens each of these links, without giving away any of our company's information:

```
I have some URLs I need to access frequently for my project at work. In this case, I cannot share our private GitHub repo information with you. But I can use variable names instead. Write me a PowerShell function that writes URL Strings in this form with the variables `$Prefix` and `$Microservice`:

# Generate-MigrationUrls.ps1

"https://github.com/my-organization/$Prefix_$Microservice",
"https://github.com/my-organization/$Prefix_$Microservice-config/foo.yaml"
"https://deployment.platform/my-namespace?name=$Microservice"
"https://build.platform/#/applications/$Microservice"
"https://monitoring.platform/{possibly_very_long_filter}params:(query:$MicroserviceBase){more_long_filter}
"https://monitoring.platform/{filter_for_different_type_of_log}params:(query:$MicroserviceBase){more_long_filter}"

I may also need to add URLs later. So make these variables parameters.
```
## Additions and Limitations

It's easy to ask ChatGPT to modify the script so that each of the URLs gets opened in FireFox, and we have a usable script which goes directly to each of the URLs we'd like.

```
function Generate-MigrationUrls {
    param(
        [string]$Prefix,
        [string]$Microservice,
        [string[]]$AdditionalUrls
    )

    $Urls = @(
        "https://github.com/my-organization/${Prefix}_${Microservice}",
        "https://github.com/my-organization/${Prefix}_${Microservice}-config/foo.yaml",
        "https://deployment.platform/my-namespace?name=$Microservice",
        "https://build.platform/#/applications/$Microservice",
        "https://monitoring.platform/{possibly_very_long_filter}params:(query:$MicroserviceBase){more_long_filter}",
        "https://monitoring.platform/{filter_for_different_type_of_log}params:(query:$MicroserviceBase){more_long_filter}"
    )

    # Add additional URLs if provided
    if ($AdditionalUrls) {
        $Urls += $AdditionalUrls
    }

    foreach ($Url in $Urls) {
        Start-Process "firefox" $Url
    }
}
```

But, opening six tabs at once puts too much of a strain on the browser. It would be nice if we were presented the services by the script as options and we could choose to open one of them. Can ChatGPT do that for us? Let's try this prompt

*"Opening six tabs at once puts too much of a strain on the browser. It would be nice if we were presented the services by the script as options and we could choose to open one of them. Can you modify this script so that each URL is present as a number and when the user types in a number only that URL is opened in FireFox?"*

Here's what it gave me:

```
function Generate-MigrationUrls {
    param(
        [string]$Prefix,
        [string]$Microservice,
        [string[]]$AdditionalUrls
    )

    $Urls = @(
        "https://github.com/my-organization/${Prefix}_${Microservice}",
        "https://github.com/my-organization/${Prefix}_${Microservice}-config/foo.yaml",
        "https://deployment.platform/my-namespace?name=$Microservice",
        "https://build.platform/#/applications/$Microservice",
        "https://monitoring.platform/{possibly_very_long_filter}params:(query:$MicroserviceBase){more_long_filter}",
        "https://monitoring.platform/{filter_for_different_type_of_log}params:(query:$MicroserviceBase){more_long_filter}"
    )

    # Add additional URLs if provided
    if ($AdditionalUrls) {
        $Urls += $AdditionalUrls
    }

    # Display URLs with corresponding numbers
    Write-Host "Choose a URL to open in Firefox:"
    for ($i = 0; $i -lt $Urls.Count; $i++) {
        Write-Host "$($i + 1). $($Urls[$i])"
    }

    # Prompt user for input
    $choice = Read-Host "Enter the number of the URL you want to open"

    # Validate user input
    if ($choice -ge 1 -and $choice -le $Urls.Count) {
        $urlToOpen = $Urls[$choice - 1]
        Start-Process "firefox" $urlToOpen
    }
    else {
        Write-Host "Invalid choice. Please enter a number between 1 and $($Urls.Count)"
    }
}
```

Which is pretty good! However, notice that it has decided to add a list of lists `$AdditionalUrls`. It may be that we do not want to implement additional URLs that way. Also, when I was working through this on my own, ChatGPT struggled with this part. Ambiguity leads to hallucinations.

## Problem Scope

Using ChatGPT to write a PowerShell script sped up the process of migrating each app by at least two fold since it let me focus on the process of migration rather than the process of opening and navigating through multiple browser interfaces. There is a temptation to continue down this path: why not map each of our task management tickets (e.g. BugZilla) to the URLs and generate them as well? Or automate the final step of filling out the Excel sheet our vendor is looking for?

There is a meme which tells a cautionary tale:

*"When there's a task that can be done manually in 10 minutes but you find a way to automate it in 10 days"*

Here, ChatGPT has brought that (hyperbolic) 10 days down to around an hour. If we have to do the task in question 20+ times in a day, then the time saved by automation is worth it. But, as we want to automate more steps and we bring in more business logic, ChatGPT's ability to reduce that 10 day figure decreases.

Always look to automate, but beware of over-automating!
