---
layout:  post
title:   "Reddit.NET console example"
summary: "A minimal example of using the Reddit API using C#."
date:    2021-02-01 21:44:00 -0800
categories: all
---

The official Reddit API instructions offer very few code examples of creating a console app.

The Reddit.NET GitHub repo documentation does not show examples of how to login before consuming their APIs.

So I created this fully working, minimal repro example of a console script in C# using .NET 5.0.

**Prerequisite**: You need to follow the instructions to create your own Reddit App, so you can obtain an AppId and a Secret:
https://github.com/reddit-archive/reddit/wiki/oauth2

_Note: These instructions only apply for a console script. To use Reddit.NET in a web app or an installed app, you need to follow different steps._

### Project file

```xml
<Project Sdk="Microsoft.NET.Sdk">

<PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net5.0</TargetFramework>
    <nullable>enable</nullable>
</PropertyGroup>

<ItemGroup>
    <PackageReference Include="Reddit" Version="1.4.0" />
</ItemGroup>

</Project>
```

### Source code

```cs
using Reddit;
using System;
using System.Diagnostics.CodeAnalysis;
using System.IO;
using System.Net;
using System.Text;
using System.Text.Json;

namespace redditcontrol
{
    class Program
    {
        private const string AppId =  "A1b2C3d4E5f6G7"; // Change me
        private const string Secret = "h1I2j3K4l5M6n7O8p9Q0r1S2t3U4v5"; // Change me

        private const string Username = "snoo"; // Change me
        private const string Password = "SnooPassw0rd!"; // Change me

        static void Main()
        {
            if (!TryGetAccessToken(out RedditToken? token))
            {
                Console.WriteLine($"Reddit token is null.");
                return;
            }
            if (string.IsNullOrEmpty(token.error))
            {
                Console.WriteLine($"Reddit token error: {token.error}");
                return;
            }

            var client = new RedditClient(
                appId: AppId,
                refreshToken: token.refresh_token,
                appSecret: Secret,
                accessToken: token.access_token);

            Reddit.Controllers.User me = client.Account.Me;

            Console.WriteLine($"Name: {me.Name}");
            Console.WriteLine($"Comment karma: {me.CommentKarma}");
            Console.WriteLine($"Link karma: {me.LinkKarma}");
        }

        private static bool TryGetAccessToken([NotNullWhen(returnValue: true)] out RedditToken? token)
        {
            HttpWebRequest request = (HttpWebRequest)WebRequest.Create("https://www.reddit.com/api/v1/access_token");
            request.PreAuthenticate = true;

            string requestAuthLogin = Convert.ToBase64String(Encoding.Default.GetBytes(AppId + ":" + Secret));
            request.Headers["Authorization"] = "Basic " + requestAuthLogin;
            request.Method = "POST";
            request.ContentType = "application/x-www-form-urlencoded";

            using Stream stream = request.GetRequestStream();
            byte[] postArray = Encoding.ASCII.GetBytes($"grant_type=password&username={Username}&password={Password}");
            stream.Write(postArray, 0, postArray.Length);

            using var reader = new StreamReader(request.GetResponse().GetResponseStream());
            string result = reader.ReadToEnd();

            token = JsonSerializer.Deserialize<RedditToken>(result);
            return token != null;
        }
    }

    class RedditToken
    {
        public string access_token { get; set; } = string.Empty;
        public string token_type { get; set; } = string.Empty;
        public int expires_in { get; set; }
        public string scope { get; set; } = string.Empty;
        public string refresh_token { get; set; } = string.Empty;

        // If error has a non-empty value, nothing else was captured
        public string error { get; set; } = string.Empty;
    }
}
```