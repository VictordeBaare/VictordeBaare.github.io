<style> .blue-border {border-left:solid 4px lightblue; padding-left:20px; border-top:dotted 4px darkcyan;} </style> <style> [class$="post"] { border-bottom:dotted 4px darkcyan; } img { max-width: 20%; height: auto; border-radius: 10%; opacity: 0.9; } </style>
Blog posts

---

# Software Composition Analysis

Open source software is incredibly popular, it is safe to say that it changed the software landscape in the last decade.
While this is incredible, it also comes with risks. Code is written by humans and yesterdays code might contain a critical vulnerabilty tomorrow.
Now how are we to mitigate these security vulnerabilties? We can hardly ask every developer to check the code which is used from third parties, especially now that a short release cycle is expected in the development process. 
Luckily there is a solution for this problem, Software Composition Analysis or SCA.

SCA is used to scan your solution for possible security vulnerabilities in your dependencies. Currently there are multiple solutions which you can use to identify and fix security vulnerabilities here are a few of the most popular ones:

- [Mend](https://docs.mend.io/)
- [Snyk](https://docs.snyk.io/products/snyk-open-source)
- [GitGuardian](https://docs.gitguardian.com/internal-repositories-monitoring/home) (GitHub)
- [GitLab](https://docs.gitlab.com/ee/user/application_security/dependency_scanning/)

Mend and Snyk can be integrated in your current DevOps process. Whereas GitLab and GitGuardian are full platforms which offer these services.

Now with adding SCA is not enough. Integrating it in your development process is crucial. These tools can highlight issues, but the development team needs to resolve them. I would advice to add the scans to your pull request process. Similar to a gated build add the SCA check to your pipeline. With this you are ahead of the issues, but we arent there yet. We also need to scan existing application which might not deploy very often anymore. So a nightly build is also crucial to get informed over new security issues in existing packages!

If you combine these two pipelines with a team which is eager to fix these kind of issues then you already tackled a set of security issues and made your software a bit more secure.