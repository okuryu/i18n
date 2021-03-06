# Pull-Requests

Es gibt zwei grundlegende Bestandteile im Pull-Request-Prozess: einen konkreten, technischen und einen mehr prozessorientierten. Der konkrete, technische Bestandteil schließt die spezifischen Details der Konfiguration Ihrer lokalen Entwicklungsumgebung mit ein, so dass Sie die eigentlichen Änderungen vornehmen können. Hier werden wir beginnen.

* [Voraussetzungen](#dependencies)
* [Einrichtung Ihrer lokalen Entwicklungsumgebung](#setting-up-your-local-environment)
  * [Schritt 1: Fork](#step-1-fork)
  * [Schritt 2: Abzweigung](#step-2-branch)
* [Der Änderungs-Prozess](#the-process-of-making-changes)
  * [Schritt 3: Code](#step-3-code)
  * [Schritt 4: Commit](#step-4-commit)
    * [Commit-Mitteilungsrichtlinien](#commit-message-guidelines)
  * [Schritt 5: Rebase](#step-5-rebase)
  * [Schritt 6: Test](#step-6-test)
    * [Testabdeckung](#test-coverage)
  * [Schritt 7: Push](#step-7-push)
  * [Schritt 8: Öffnen des Pull-Requests](#step-8-opening-the-pull-request)
  * [Schritt 9: Diskussion und Update](#step-9-discuss-and-update)
    * [Workflow von Genehmigung und Änderungsanfrage](#approval-and-request-changes-workflow)
  * [Schritt 10: Landung](#step-10-landing)
* [Überprüfen der Pull-Requests](#reviewing-pull-requests)
  * [Überprüfen Sie eins nach dem anderen](#review-a-bit-at-a-time)
  * [Beachten Sie die Person hinter dem Code](#be-aware-of-the-person-behind-the-code)
  * [Respektieren Sie eine Mindest-Wartezeit für Kommentare](#respect-the-minimum-wait-time-for-comments)
  * [Vergessene oder ins Stocken geratene Pull-Requests](#abandoned-or-stalled-pull-requests)
  * [Genehmigung einer Änderung](#approving-a-change)
  * [Akzeptieren Sie, dass es verschiedene Meinungen darüber gibt, was zu Node.js gehört](#accept-that-there-are-different-opinions-about-what-belongs-in-nodejs)
  * [Leistung ist nicht alles](#performance-is-not-everything)
  * [Fortlaufende Integrations-Prüfung](#continuous-integration-testing)
* [Weitere Informationen](#additional-notes)
  * [Commit-Squashing](#commit-squashing)
  * [Genehmigungen für Ihre Pull-Requests erhalten](#getting-approvals-for-your-pull-request)
  * [CI-Test](#ci-testing)
  * [Warten, bis der Pull-Request gelandet wurde](#waiting-until-the-pull-request-gets-landed)
  * [Check Out the Collaborator's Guide](#check-out-the-collaborators-guide)

## Voraussetzungen

Node.js has several bundled dependencies in the *deps/* and the *tools/* directories that are not part of the project proper. Änderungen in den Dateien in jenen Verzeichnissen sollten an deren jeweilige Projekte gesendet werden. Senden Sie keinen Patch an Node.js. Wir können solche Patches nicht akzeptieren.

Im Zweifelsfall öffnen Sie einen "Problemfall" im [Problem-Tracker](https://github.com/nodejs/node/issues/) oder kontaktieren Sie einen der [Projekt-Mitarbeiter](https://github.com/nodejs/node/#current-project-team-members). Node.js hat zwei IRC-Channel: [#Node.js](https://webchat.freenode.net/?channels=node.js) für allgemeine Hilfe und Fragen sowie [#Node-dev](https://webchat.freenode.net/?channels=node-dev) spezifisch für die Entwicklung des Node.js-Core.

## Einrichtung Ihrer lokalen Entwicklungsumgebung

Um anzufangen, müssen Sie `git` lokal installiert haben. Abhängig von Ihrem Betriebssystem, gibt es auch eine Anzahl weiterer erforderlicher Voraussetzungen. Diese sind im Detail in der [Bauanleitung](../../../BUILDING.md) beschrieben.

Sobald Sie `git` haben und sicher sind, dass Sie alle erforderlichen Voraussetzungen haben, ist es an der Zeit, einen Fork zu erschaffen.

Before getting started, it is recommended to configure `git` so that it knows who you are:

```text
$ git config --global user.name "J. Random User"
$ git config --global user.email "j.random.user@example.com"
```
Bitte stellen Sie sicher, dass diese lokale E-Mail-Adresse auch zu Ihrer [GitHub-E-Mail-Liste](https://github.com/settings/emails) hinzugefügt wurde, so dass Ihre Commits ordnungsgemäß mit Ihrem Account verknüpft werden und Sie zum "Teilnehmer" befördert werden, sobald Ihr erster Commit gelandet wurde.

### Schritt 1: Fork

Forken Sie das Projekt [on Github](https://github.com/nodejs/node) und duplizieren Sie Ihren Fork lokal.

```text
$ git clone git@github.com:username/node.git
$ cd node
$ git remote add upstream https://github.com/nodejs/node.git
$ git fetch upstream
```

### Schritt 2: Abzweigung

Um Ihre Entwicklungsumgebung so organisiert wie möglich zu halten, hat es sich als praktisch erwiesen, lokale Abzweigungen zu schaffen, in denen man arbeitet. Diese sollten auch außerhalb der `Master`-Abzweigung geschaffen werden.

```text
$ git checkout -b my-branch -t upstream/master
```

## Der Änderungs-Prozess

### Schritt 3: Code

The vast majority of Pull Requests opened against the `nodejs/node` repository includes changes to either the C/C++ code contained in the `src` directory, the JavaScript code contained in the `lib` directory, the documentation in `docs/api` or tests within the `test` directory.

Wenn Sie Code verändern, stellen Sie bitte sicher, dass Sie `make lint` von Zeit zu Zeit ausführen, um zu gewährleisten, dass die Änderungen den Node.js-Code-Stilrichtlinien entsprechen.

Jede Dokumentation, die Sie verfassen (inklusive Coding-Kommentare und die API-Dokumentation), sollte den [Stilrichtlinien](../../STYLE_GUIDE.md) entsprechen. Code-Beispiele, die in den API-Dokumenten enthalten sind, werden auch überprüft, wenn `make lint` ausgeführt wird (oder `vcbuild.bat lint` unter Windows).

Bei Einreichung von C++-Code, sollten Sie vielleicht einen Blick auf die [C++-Stilrichtlinien](../../../CPP_STYLE_GUIDE.md) werfen.

### Schritt 4: Commit

Es ist empfohlene gängige Praxis, dass Sie Ihre Änderungen so logisch wie möglich innerhalb individueller Commits gruppieren. Es gibt keine Grenze für die Anzahl an Commits, die ein einzelner Pull-Request haben kann, und viele Teilnehmer finden es einfacher, die Änderungen zu überprüfen, die über mehrere Commits verteilt werden.

```text
$ git add my/changed/files
$ git commit
```

Beachten Sie, dass mehrere Commits oft gesquasht werden, wenn sie gelandet werden (siehe Anmerkungen über das [Commit-Squashing](#commit-squashing)).

#### Commit-Mitteilungsrichtlinien

Eine gute Commit-Mitteilung sollte beschreiben, was sich geändert hat und warum.

1. Die erste Zeile sollte:
   - contain a short description of the change (preferably 50 characters or less, and no more than 72 characters)
   - vollständig in Kleinbuchstaben verfasst sein, mit Ausnahme von Eigennamen, Akronymen und Wörtern, die auf Code verweisen, wie Funktionen- und Variablennamen
   - sollten einen Präfix mit dem Namen des geänderten Subsystems enthalten und mit einem Imperativ-Verb beginnen. Schauen Sie sich die Ausgabe von `git log --oneline files/you/changed` an, um herauszufinden, welche Subsysteme von Ihren Änderungen betroffen sind.

   Beispiele:
   - `net: add localAddress and localPort to Socket`
   - `src: fix typos in node_lttng_provider.h`


2. Lassen Sie die zweite Zeile leer.
3. Wrap all other lines at 72 columns.

4. Falls Ihr Patch ein aktuelles Problem behebt, können Sie einen Verweis darauf am Ende des Logs einfügen. Benutzen Sie den `Fixes:`-Präfix und die vollständige URL des Problemfalls. Für andere Verweise nutzen Sie `Refs:`.

   Beispiele:
   - `Fixes: https://github.com/nodejs/node/issues/1337`
   - `Refs: http://eslint.org/docs/rules/space-in-parens.html`
   - `Refs: https://github.com/nodejs/node/pull/3615`

5. Falls Ihr Commit eine wesentliche Änderung einführt (`semver-major`), sollte die Mitteilung eine Erklärung zum Grund für die wesentliche Änderung enthalten, welche Situation die wesentliche Änderung auslösen könnte und was genau geändert wurde.

Breaking changes will be listed in the wiki with the aim to make upgrading easier.  Please have a look at [Breaking Changes](https://github.com/nodejs/node/wiki/Breaking-changes-between-v4-LTS-and-v6-LTS) for the level of detail that's suitable.

Beispiel-Commit-Mitteilung:

```txt
subsystem: explain the commit in one line

Body of commit message is a few lines of text, explaining things
in more detail, possibly giving some background about the issue
being fixed, etc.

The body of the commit message can be several paragraphs, and
please do proper word-wrap and keep columns shorter than about
72 characters or so. That way, `git log` will show things nicely even when it is indented.

Fixes: https://github.com/nodejs/node/issues/1337
Refs: http://eslint.org/docs/rules/space-in-parens.html
```

Falls Sie mit der Teilnahme an Node.js noch nicht vertraut sind, geben Sie bitte Ihr Bestes, um diese Richtlinien einzuhalten, aber seien Sie unbesorgt, falls etwas schief läuft. Einer der vorhandenen Mitwirkenden wird dabei helfen, die Dinge zu richten und der Teilnehmer, welcher den Pull-Request landet, wird sicherstellen, dass alles den Projekt-Richtlinien entspricht.

Siehe [core-validate-commit](https://github.com/evanlucas/core-validate-commit) - Ein Hilfsmittel, das sicherstellt, dass Commits den Commit-Formatierungs-Richtlinien entsprechen.

### Schritt 5: Rebase

Es hat sich als praktisch erwiesen, dass es eine gute Idee ist, `git rebase` zu nutzen (nicht `git merge`), sobald Sie Ihre Änderungen eingereicht haben, um Ihre Arbeit mit dem Haupt-Repository zu synchronisieren.

```text
$ git fetch upstream
$ git rebase upstream/master
```

Dies stellt sicher, dass Ihre Arbeits-Abzweigung die neuesten Änderungen vom `nodejs/node`-Master enthält.

### Schritt 6: Test

Bugfixes und Features sollten immer getestet werden. Ein [Guide zum Schreiben von Tests in Node.js](../writing-tests.md) wird zur Verfügung gestellt, um den Prozess einfacher zu gestalten. Sich andere Tests anzusehen, um zu erfahren, wie diese strukturiert sein sollten, kann ebenfalls helfen.

Das `test`-Verzeichnis innerhalb des `nodejs/node`-Repositories ist komplex und es ist oftmals nicht klar, wo eine neue Testdatei abgelegt werden soll. Im Zweifel fügen Sie neue Tests zum `test/parallel/`-Verzeichnis hinzu und der richtige Ort wird später lokalisiert.

Bevor Sie Ihre Änderungen in einem Pull-Request einreichen, starten Sie immer die komplette Node.js-Test-Suite. Um die Tests (inklusive Code-Linting) unter Unix / macOS auszuführen:

```text
$ ./configure && make -j4 test
```

Und unter Windows:

```text
> vcbuild test
```

(Siehe die [Bauanleitung](../../../BUILDING.md) für weitere Details.)

Stellen Sie sicher, dass der Lint keinen Fehler meldet und dass alle Tests bestanden werden. Bitte reichen Sie keine Patches ein, die einem Test nicht standhalten.

Falls Sie den Lint ohne Tests ausführen wollten, benutzen Sie `make lint` / `vcbuild lint`. Dies wird sowohl JavaScript- als auch C++-Linting ausführen.

Falls Sie Tests updaten und nur einen Einzeltest zur Überprüfung ausführen wollen:

```text
$ python tools/test.py -J --mode=release parallel/test-stream2-transform
```

Sie können die gesamte Testsuite für ein bestehendes Subsystem ausführen, indem Sie den Namen eines Subsystems angeben:

```text
$ python tools/test.py -J --mode=release child-process
```

Wenn Sie die anderen Optionen checken wollen, wenden Sie sich bitte an die Hilfe durch Nutzung der `--help`-Option

```text
$ python tools/test.py --help
```

Sie können Tests normalerweise direkt mit der Node ausführen:

```text
$ ./node ./test/parallel/test-stream2-transform.js
```

Denken Sie an das Re-Kompilieren mit `make -j4` zwischen den Testläufen, falls Sie Code in den `lib`- oder `src`-Verzeichnissen ändern.

#### Testabdeckung

Es wäre gut sicherzustellen, dass jeglicher Code, den Sie hinzufügen oder ändern, durch Tests abgedeckt wird. Sie erreichen dies durch Ausführung der Test-Suite mit eingeschalteter Abdeckung:

```text
$ ./configure --coverage && make coverage
```

Ein detailierter Abdeckungs-Bericht wird in `coverage/index.html` erstellt für die JavaScript-Abdeckung und in `coverage/cxxcoverage.html` für die C++-Abdeckung.

_Beachten Sie, dass die Generierung eines Testabdeckungs-Berichtes mehrere Minuten dauern kann._

Um die Abdeckung für einen Teil der Tests einzusammeln, können Sie die `CI_JS_SUITES`- und `CI_NATIVE_SUITES`-Variablen festlegen:

```text
$ CI_JS_SUITES=child-process CI_NATIVE_SUITES= make coverage
```

Der oben genannte Befehl führt Tests für das `child-process`-Subsystem aus und liefert den sich hieraus ergebenden Abdeckungs-Bericht aus.

Das Ausführen von Tests mit Abdeckung wird verschiedene Verzeichnisse und Dateien erzeugen und verändern. Um danach aufzuräumen, führen Sie aus:

```text
make coverage-clean
./configure && make -j4.
```

### Schritt 7: Push

Once you are sure your commits are ready to go, with passing tests and linting, begin the process of opening a Pull Request by pushing your working branch to your fork on GitHub.

```text
$ git push origin my-branch
```

### Schritt 8: Öffnen des Pull-Requests

From within GitHub, opening a new Pull Request will present you with a template that should be filled out:

```markdown
<!--
Thank you for your Pull Request. Please provide a description above and review
the requirements below.

Bug fixes and new features should include tests and possibly benchmarks.

Contributors guide: https://github.com/nodejs/node/blob/master/CONTRIBUTING.md
-->

#### Checklist
<!-- Remove items that do not apply. For completed items, change [ ] to [x]. -->

- [ ] `make -j4 test` (UNIX), or `vcbuild test` (Windows) passes
- [ ] tests and/or benchmarks are included
- [ ] documentation is changed or added
- [ ] commit message follows [commit guidelines](https://github.com/nodejs/node/blob/master/doc/guides/contributing/pull-requests.md#commit-message-guidelines)

#### Affected core subsystem(s)
<!-- Provide affected core subsystem(s) (like doc, cluster, crypto, etc). -->
```

Please try to do your best at filling out the details, but feel free to skip parts if you're not sure what to put.

Once opened, Pull Requests are usually reviewed within a few days.

### Step 9: Discuss and update

You will probably get feedback or requests for changes to your Pull Request. This is a big part of the submission process so don't be discouraged! Some contributors may sign off on the Pull Request right away, others may have more detailed comments or feedback. This is a necessary part of the process in order to evaluate whether the changes are correct and necessary.

To make changes to an existing Pull Request, make the changes to your local branch, add a new commit with those changes, and push those to your fork. GitHub will automatically update the Pull Request.

```text
$ git add my/changed/files
$ git commit
$ git push origin my-branch
```

It is also frequently necessary to synchronize your Pull Request with other changes that have landed in `master` by using `git rebase`:

```text
$ git fetch --all
$ git rebase origin/master
$ git push --force-with-lease origin my-branch
```

**Important:** The `git push --force-with-lease` command is one of the few ways to delete history in `git`. Before you use it, make sure you understand the risks. If in doubt, you can always ask for guidance in the Pull Request or on [IRC in the #node-dev channel](https://webchat.freenode.net?channels=node-dev&uio=d4).

If you happen to make a mistake in any of your commits, do not worry. You can amend the last commit (for example if you want to change the commit log).

```text
$ git add any/changed/files
$ git commit --amend
$ git push --force-with-lease origin my-branch
```

There are a number of more advanced mechanisms for managing commits using `git rebase` that can be used, but are beyond the scope of this guide.

Feel free to post a comment in the Pull Request to ping reviewers if you are awaiting an answer on something. If you encounter words or acronyms that seem unfamiliar, refer to this [glossary](https://sites.google.com/a/chromium.org/dev/glossary).

#### Workflow von Genehmigung und Änderungsanfrage

All Pull Requests require "sign off" in order to land. Whenever a contributor reviews a Pull Request they may find specific details that they would like to see changed or fixed. These may be as simple as fixing a typo, or may involve substantive changes to the code you have written. While such requests are intended to be helpful, they may come across as abrupt or unhelpful, especially requests to change things that do not include concrete suggestions on *how* to change them.

Try not to be discouraged. If you feel that a particular review is unfair, say so, or contact one of the other contributors in the project and seek their input. Often such comments are the result of the reviewer having only taken a short amount of time to review and are not ill-intended. Such issues can often be resolved with a bit of patience. That said, reviewers should be expected to be helpful in their feedback, and feedback that is simply vague, dismissive and unhelpful is likely safe to ignore.

### Schritt 10: Landung

In order to land, a Pull Request needs to be reviewed and [approved](#getting-approvals-for-your-pull-request) by at least one Node.js Collaborator and pass a [CI (Continuous Integration) test run](#ci-testing). After that, as long as there are no objections from other contributors, the Pull Request can be merged. If you find your Pull Request waiting longer than you expect, see the [notes about the waiting time](#waiting-until-the-pull-request-gets-landed).

When a collaborator lands your Pull Request, they will post a comment to the Pull Request page mentioning the commit(s) it landed as. GitHub often shows the Pull Request as `Closed` at this point, but don't worry. If you look at the branch you raised your Pull Request against (probably `master`), you should see a commit with your name on it. Congratulations and thanks for your contribution!

## Überprüfen der Pull-Requests

All Node.js contributors who choose to review and provide feedback on Pull Requests have a responsibility to both the project and the individual making the contribution. Reviews and feedback must be helpful, insightful, and geared towards improving the contribution as opposed to simply blocking it. If there are reasons why you feel the PR should not land, explain what those are. Do not expect to be able to block a Pull Request from advancing simply because you say "No" without giving an explanation. Be open to having your mind changed. Be open to working with the contributor to make the Pull Request better.

Reviews that are dismissive or disrespectful of the contributor or any other reviewers are strictly counter to the [Code of Conduct](https://github.com/nodejs/admin/blob/master/CODE_OF_CONDUCT.md).

When reviewing a Pull Request, the primary goals are for the codebase to improve and for the person submitting the request to succeed. Even if a Pull Request does not land, the submitters should come away from the experience feeling like their effort was not wasted or unappreciated. Every Pull Request from a new contributor is an opportunity to grow the community.

### Überprüfen Sie eins nach dem anderen.

Do not overwhelm new contributors.

It is tempting to micro-optimize and make everything about relative performance, perfect grammar, or exact style matches. Do not succumb to that temptation.

Focus first on the most significant aspects of the change:

1. Does this change make sense for Node.js?
2. Does this change make Node.js better, even if only incrementally?
3. Are there clear bugs or larger scale issues that need attending to?
4. Is the commit message readable and correct? If it contains a breaking change is it clear enough?

When changes are necessary, *request* them, do not *demand* them, and do not assume that the submitter already knows how to add a test or run a benchmark.

Specific performance optimization techniques, coding styles and conventions change over time. The first impression you give to a new contributor never does.

Nits (requests for small changes that are not essential) are fine, but try to avoid stalling the Pull Request. Most nits can typically be fixed by the Node.js Collaborator landing the Pull Request but they can also be an opportunity for the contributor to learn a bit more about the project.

It is always good to clearly indicate nits when you comment: e.g. `Nit: change foo() to bar(). But this is not blocking.`

### Beachten Sie die Person hinter dem Code

Be aware that *how* you communicate requests and reviews in your feedback can have a significant impact on the success of the Pull Request. Yes, we may land a particular change that makes Node.js better, but the individual might just not want to have anything to do with Node.js ever again. The goal is not just having good code.

### Respektieren Sie eine Mindest-Wartezeit für Kommentare

There is a minimum waiting time which we try to respect for non-trivial changes, so that people who may have important input in such a distributed project are able to respond.

For non-trivial changes, Pull Requests must be left open for *at least* 48 hours during the week, and 72 hours on a weekend. In most cases, when the PR is relatively small and focused on a narrow set of changes, these periods provide more than enough time to adequately review. Sometimes changes take far longer to review, or need more specialized review from subject matter experts. When in doubt, do not rush.

Trivial changes, typically limited to small formatting changes or fixes to documentation, may be landed within the minimum 48 hour window.

### Vergessene oder ins Stocken geratene Pull-Requests

If a Pull Request appears to be abandoned or stalled, it is polite to first check with the contributor to see if they intend to continue the work before checking if they would mind if you took it over (especially if it just has nits left). When doing so, it is courteous to give the original contributor credit for the work they started (either by preserving their name and email address in the commit log, or by using an `Author:` meta-data tag in the commit.

### Genehmigung einer Änderung

Any Node.js core Collaborator (any GitHub user with commit rights in the `nodejs/node` repository) is authorized to approve any other contributor's work. Collaborators are not permitted to approve their own Pull Requests.

Collaborators indicate that they have reviewed and approve of the changes in a Pull Request either by using GitHub's Approval Workflow, which is preferred, or by leaving an `LGTM` ("Looks Good To Me") comment.

When explicitly using the "Changes requested" component of the GitHub Approval Workflow, show empathy. That is, do not be rude or abrupt with your feedback and offer concrete suggestions for improvement, if possible. If you're not sure *how* a particular change can be improved, say so.

Most importantly, after leaving such requests, it is courteous to make yourself available later to check whether your comments have been addressed.

If you see that requested changes have been made, you can clear another collaborator's `Changes requested` review.

Change requests that are vague, dismissive, or unconstructive may also be dismissed if requests for greater clarification go unanswered within a reasonable period of time.

If you do not believe that the Pull Request should land at all, use `Changes requested` to indicate that you are considering some of your comments to block the PR from landing. When doing so, explain *why* you believe the Pull Request should not land along with an explanation of what may be an acceptable alternative course, if any.

### Akzeptieren Sie, dass es verschiedene Meinungen darüber gibt, was zu Node.js gehört

Opinions on this vary, even among the members of the Technical Steering Committee.

One general rule of thumb is that if Node.js itself needs it (due to historic or functional reasons), then it belongs in Node.js. For instance, `url` parsing is in Node.js because of HTTP protocol support.

Also, functionality that either cannot be implemented outside of core in any reasonable way, or only with significant pain.

It is not uncommon for contributors to suggest new features they feel would make Node.js better. These may or may not make sense to add, but as with all changes, be courteous in how you communicate your stance on these. Comments that make the contributor feel like they should have "known better" or ridiculed for even trying run counter to the [Code of Conduct](https://github.com/nodejs/admin/blob/master/CODE_OF_CONDUCT.md).

### Leistung ist nicht alles

Node.js has always optimized for speed of execution. If a particular change can be shown to make some part of Node.js faster, it's quite likely to be accepted. Claims that a particular Pull Request will make things faster will almost always be met by requests for performance [benchmark results](../writing-and-running-benchmarks.md) that demonstrate the improvement.

That said, performance is not the only factor to consider. Node.js also optimizes in favor of not breaking existing code in the ecosystem, and not changing working functional code just for the sake of changing.

If a particular Pull Request introduces a performance or functional regression, rather than simply rejecting the Pull Request, take the time to work *with* the contributor on improving the change. Offer feedback and advice on what would make the Pull Request acceptable, and do not assume that the contributor should already know how to do that. Be explicit in your feedback.

### Fortlaufende Integrations-Prüfung

All Pull Requests that contain changes to code must be run through continuous integration (CI) testing at [https://ci.nodejs.org/](https://ci.nodejs.org/).

Only Node.js core Collaborators with commit rights to the `nodejs/node` repository may start a CI testing run. The specific details of how to do this are included in the new Collaborator [Onboarding guide](../../onboarding.md).

Ideally, the code change will pass ("be green") on all platform configurations supported by Node.js (there are over 30 platform configurations currently). This means that all tests pass and there are no linting errors. In reality, however, it is not uncommon for the CI infrastructure itself to fail on specific platforms or for so-called "flaky" tests to fail ("be red"). It is vital to visually inspect the results of all failed ("red") tests to determine whether the failure was caused by the changes in the Pull Request.

## Weitere Informationen

### Commit-Squashing

When the commits in your Pull Request land, they may be squashed into one commit per logical change. Metadata will be added to the commit message (including links to the Pull Request, links to relevant issues, and the names of the reviewers). The commit history of your Pull Request, however, will stay intact on the Pull Request page.

For the size of "one logical change", [0b5191f](https://github.com/nodejs/node/commit/0b5191f15d0f311c804d542b67e2e922d98834f8) can be a good example. It touches the implementation, the documentation, and the tests, but is still one logical change. All tests should always pass when each individual commit lands on the master branch.

### Getting Approvals for Your Pull Request

A Pull Request is approved either by saying LGTM, which stands for "Looks Good To Me", or by using GitHub's Approve button. GitHub's Pull Request review feature can be used during the process. For more information, check out [the video tutorial](https://www.youtube.com/watch?v=HW0RPaJqm4g) or [the official documentation](https://help.github.com/articles/reviewing-changes-in-pull-requests/).

After you push new changes to your branch, you need to get approval for these new changes again, even if GitHub shows "Approved" because the reviewers have hit the buttons before.

### CI-Test

Every Pull Request needs to be tested to make sure that it works on the platforms that Node.js supports. This is done by running the code through the CI system.

Only a Collaborator can start a CI run. Usually one of them will do it for you as approvals for the Pull Request come in. If not, you can ask a Collaborator to start a CI run.

### Warten, bis der Pull-Request gelandet wurde

A Pull Request needs to stay open for at least 48 hours (72 hours on a weekend) from when it is submitted, even after it gets approved and passes the CI. This is to make sure that everyone has a chance to weigh in. If the changes are trivial, collaborators may decide it doesn't need to wait. A Pull Request may well take longer to be merged in. All these precautions are important because Node.js is widely used, so don't be discouraged!

### Check Out the Collaborator's Guide

If you want to know more about the code review and the landing process, you can take a look at the [collaborator's guide](https://github.com/nodejs/node/blob/master/COLLABORATOR_GUIDE.md).
