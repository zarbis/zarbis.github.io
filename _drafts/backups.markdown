# The trends

Through my DevOps journey I've noticed one trend, common for traditional "Ops duties". To list few of them and classical approach for each:

* Dependencies: some Ops guy would set up server in a way that satisfies dependencies of code being deployed to it.
* Exposure: after initial deployment networking guy would perform some sort of magic and that application will be accessible to users on certain domain and route.
* Logging and Monitoring: after some time we would decide that we need more insights of how our application works and point different systems to scrape metrics and perform checks on it.

# The state

Traditionally those aspects of app life cycle were performed separately from "main act" of code deployment. Both in terms of time and people. We went long path towards tightly coupling those aspects to app itself. Yes, the rare kind of tight coupling that is actually good. Lets take a look at current state of affairs:

* Dependencies: are now declared in Dockerfile by developer to form immutable self-contained unit of functionality.
* Exposure: "request for exposure" is called Ingress resource, if we are speaking Kubernetes.
* Monitoring: "i want to be monitored" is most likely Prometheus annotation in one of your app's k8s resources.
* Logging: well, just be a container. Log forwarding was probably set up long time ago during cluster setup.

All those things among many others are probably part of Helm chart that sits happily alongside actual application code. So when we want to fully deploy our application we are moving from depending on collaborative actions of multiple people to self-contained application definitions. Those definitions can be declared by single person who knows the app the best. Previously it would be impossible for developer to grasp all those different aspects of his application, but nowadays "Ops duty" is not only in maintaining "raw infrastructure", but also providing platform for development, that hides complexity of infrastructure and provides simple interface to it. As a result, developer doesn't need to be an expert in all those fields, just familiarity with 5 or so concepts and several lines of YAML are enough.

This shift in approach made developers much more self-sufficient and admins' mundane chores are at historical minimum. There is no more Chinese telephone problem because devs don't need to convey "need for dependencies/exposure/monitoring/etc" to Ops via natural language in documentation and tickets.

# The exception

However one aspect of Ops duties remains. Those are backups. We still to this day enable backups as afterthought in an ad-hoc fashion. And there are good reasons for it.

# Why so?

First of all, backups are super crucial, we can't take them lightly. Incorrectly configured ingress or monitoring, while might be disruptive to our users, is immediately apparent. On the other hand, incorrectly configured backup procedure uncovers it's faulty nature only when we try to restore them and something goes wrong, in other words: when it's too late.

Second reason is developer's unfamiliarity with concept of backups. They inevitably deal with dependencies during development process and only replicate more or less the same steps in Dockerfile. They also deal with application routing in daily basis, so Ingress resource definition is also some sort of repetition of routes declaration from source code. Monitoring is a bit of a stretch, since it's not an integral part of development process, but whitebox monitoring is basically writing additional code, like unit tests, nothing new for a good developer.

The stark contrast is in backups. Typical developer's workflow would at most involve installing database of choice and seeding it with test data.
