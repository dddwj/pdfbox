#Variable definition.
BASE_VERSION=1.0.1
SRC=$(SD_SOURCE_DIR)
GIT_BINARY=$(shell command -v git || echo "/home/y/bin64/git")
MVN_BINARY=$(shell command -v mvn || echo "/home/y/bin/mvn")
CHANGE_LOG_FILE=$(SRC)/ci/CHANGELOG
BRANCH ?= pdfbox_upgrade

pr: pr_build

component: generate_buildinfo backup_pom release_prepare switch_tag generate_changelog component_build

generate_buildinfo:
	$(eval BUILD_NO += $(shell expr `$(GIT_BINARY) tag -l TP-T-$(BASE_VERSION)* | cut -d. -f4 | sort -nr | head -1 | sed -e 's/\///'` + 1))
	$(eval PREV_BUILD_NO += $(shell expr `$(GIT_BINARY) tag -l TP-T-$(BASE_VERSION)* | cut -d. -f4 | sort -nr | head -1 | sed -e 's/\///'`))

backup_pom:
	mkdir -p $(SRC)/pomfiles
	cd $(SRC); for POM in `find . -name pom.xml`; do cp -f --parents $$POM pomfiles/; done

release_prepare:
	cd $(SRC); $(MVN_BINARY) release:prepare --batch-mode -Dmaven.repo.local=$(SD_SOURCE_DIR)/.m2 -DautoVersionSubmodules=true -Dtag=TP-T-$(BASE_VERSION).$(BUILD_NO) -DreleaseVersion=$(BASE_VERSION).$(BUILD_NO) -DdevelopmentVersion=$(BASE_VERSION)-SNAPSHOT -DpreparationGoals=clean -DpushChanges=false -Darguments="-Dmaven.test.skip=true"; \
	if test $$? != 0 ; then \
		echo ERROR: Maven release prepare failed, reverting back POM files if modified; \
		cp -Rf pomfiles/* .; \
		$(GIT_BINARY) commit -am "Release Prepare failure, reverting back POM files changes"; $(GIT_BINARY) pull origin $(BRANCH); $(GIT_BINARY) push origin $(BRANCH); \
		exit 1; \
	fi

publish_tag: generate_buildinfo backup_pom release_prepare switch_tag update_files
	cd $(SRC); $(GIT_BINARY) push origin TP-T-$(BASE_VERSION).$(BUILD_NO)

switch_tag:
	$(GIT_BINARY) checkout -b TP-$(BASE_VERSION).$(BUILD_NO) TP-T-$(BASE_VERSION).$(BUILD_NO)

generate_changelog:
	cd $(SRC)/ci; $(GIT_BINARY) log TP-T-$(BASE_VERSION).$(PREV_BUILD_NO)..TP-T-$(BASE_VERSION).$(BUILD_NO) --pretty=format:"%an: %s - %b" --first-parent -- $(SD_SOURCE_DIR) | { grep -v by-tortuga: || true; } > $(CHANGE_LOG_FILE) && echo >> $(CHANGE_LOG_FILE)
	cat $(CHANGE_LOG_FILE) | sed 's/$$/<\/br>/g' | sed 's/\[[Bb][Uu][Gg] *\([0-9]*\)\]/<a href="http:\/\/bug.corp.yahoo.com\/show_bug.cgi?id=\1">Bug \1<\/a>/g' | sed 's/Change-Id: \([a-zA-Z0-9]*\)/Change-Id: <a href="https:\/\/gerrit.comms.corp.yahoo.com\/#q,\1,n,z">\1<\/a>/g' > ../CHANGELOG.html

component_build:
	cd $(SRC); $(GIT_BINARY) add $(CHANGE_LOG_FILE); $(GIT_BINARY) commit -am "Updating changelog before creating release package"
	cd $(SRC); $(MVN_BINARY) clean deploy -T 1C -Dmaven.repo.local=$(SD_SOURCE_DIR)/.m2

pr_build:
	cd $(SRC); $(MVN_BINARY) clean install -T 1C -Dmaven.repo.local=$(SD_SOURCE_DIR)/.m2

update_files:
	cd $(SRC); cp -Rf pomfiles/* .
	cd $(SRC); $(GIT_BINARY) commit -am "Updating pom files for next build iteration"; $(GIT_BINARY) push origin TP-$(BASE_VERSION).$(BUILD_NO)
