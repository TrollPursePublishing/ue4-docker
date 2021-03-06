{% if combine %}
FROM source as builder
{% else %}
ARG NAMESPACE
ARG TAG
ARG PREREQS_TAG
FROM ${NAMESPACE}/ue4-source:${TAG}-${PREREQS_TAG} AS builder
{% endif %}

# Remove the .git directory to disable UBT `git status` calls and speed up the build process
RUN rm -rf /home/ue4/UnrealEngine/.git

# Set the changelist number in Build.version to ensure our Build ID is generated correctly
COPY set-changelist.py /tmp/set-changelist.py
RUN python3 /tmp/set-changelist.py /home/ue4/UnrealEngine/Engine/Build/Build.version

{% if (not disable_all_patches) and (not disable_opengl_patch) %}
# Enable the OpenGL RHI for Engine versions where it is present but deprecated
COPY enable-opengl.py /tmp/enable-opengl.py
RUN python3 /tmp/enable-opengl.py /home/ue4/UnrealEngine/Engine/Config/BaseEngine.ini
{% endif %}

{% if (not disable_all_patches) and (not disable_buildgraph_patches) %}
COPY patch-filters-xml.py /tmp/patch-filters-xml.py
RUN python3 /tmp/patch-filters-xml.py /home/ue4/UnrealEngine/Engine/Build/InstalledEngineFilters.xml

# Patch the default settings in InstalledEngineBuild.xml and increase the output verbosity of the DDC generation step
COPY patch-build-graph.py /tmp/patch-build-graph.py
RUN python3 /tmp/patch-build-graph.py /home/ue4/UnrealEngine/Engine/Build/InstalledEngineBuild.xml /home/ue4/UnrealEngine/Engine/Build/Build.version
{% endif %}

# Ensure UBT is built before we create the Installed Build, since Build.sh explicitly sets the
# target .NET framework version, whereas InstalledEngineBuild.xml just uses the system default,
# which can result in errors when running the built UBT due to the wrong version being targeted
RUN ./Engine/Build/BatchFiles/Linux/Build.sh UnrealHeaderTool Linux Development -SkipBuild

# Create an Installed Build of the Engine
ARG BUILDGRAPH_ARGS
WORKDIR /home/ue4/UnrealEngine
RUN ./Engine/Build/BatchFiles/RunUAT.sh BuildGraph -target="Make Installed Build Linux" -script=Engine/Build/InstalledEngineBuild.xml -set:HostPlatformOnly=true ${BUILDGRAPH_ARGS} {{ buildgraph_args }} && \
	rm -R -f /home/ue4/UnrealEngine/LocalBuilds/InstalledDDC

# Determine if we are removing debug symbols and/or template projects in order to reduce the final container image size
COPY exclude-components.py /tmp/exclude-components.py
ARG EXCLUDE_DEBUG
ARG EXCLUDE_TEMPLATES
RUN python3 /tmp/exclude-components.py /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux $EXCLUDE_DEBUG $EXCLUDE_TEMPLATES

{% if (not disable_all_patches) and (not disable_target_patches) %}
# Ensure Client and Server targets have their `PlatformType` field set correctly in BaseEngine.ini
COPY fix-targets.py /tmp/fix-targets.py
RUN python3 /tmp/fix-targets.py /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux/Engine/Config/BaseEngine.ini
{% endif %}

{% if (not disable_all_patches) and (not disable_unrealpak_copy) %}
# Some versions of the Engine fail to include UnrealPak in the Installed Build, so copy it manually
RUN cp ./Engine/Binaries/Linux/UnrealPak ./LocalBuilds/Engine/Linux/Engine/Binaries/Linux/UnrealPak
{% endif %}

{% if (not disable_all_patches) and (not disable_toolchain_copy) %}
# Ensure the bundled toolchain included in 4.20.0 and newer is copied to the Installed Build
COPY --chown=ue4:ue4 copy-toolchain.py /tmp/copy-toolchain.py
RUN python3 /tmp/copy-toolchain.py /home/ue4/UnrealEngine
{% endif %}

# Copy the Installed Build into a clean image, discarding the source build
{% if combine %}
FROM prerequisites as minimal
{% else %}
ARG NAMESPACE
FROM ${NAMESPACE}/ue4-build-prerequisites:${PREREQS_TAG}
{% endif %}

# Copy the Installed Build files from the builder image
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux /home/ue4/UnrealEngine
WORKDIR /home/ue4/UnrealEngine

{% if not disable_labels %}
# Add labels to the built image to identify which components (if any) were excluded from the build that it contains
# (Note that we need to redeclare the relevant ARG directives here because they are scoped to each individual stage in a multi-stage build)
ARG EXCLUDE_DEBUG
ARG EXCLUDE_TEMPLATES
LABEL com.adamrehn.ue4-docker.excluded.debug=${EXCLUDE_DEBUG}
LABEL com.adamrehn.ue4-docker.excluded.templates=${EXCLUDE_TEMPLATES}
{% endif %}

# Perform first-run setup for Mono, UnrealBuildTool and AutomationTool, which makes it possible to build Unreal projects and plugins as users other than `ue4`
# (Note that this will only work with 4.26.0 and newer, older Engine versions will always require write access to `/home/ue4/UnrealEngine`)
# See the comments on this issue for details, including the need to ensure $HOME is set correctly: <https://github.com/adamrehn/ue4-docker/issues/141>
RUN ./Engine/Build/BatchFiles/Linux/Build.sh UnrealHeaderTool Linux Development -SkipBuild && \
	mkdir -p ./Engine/Programs/AutomationTool/Saved && \
	chmod a+rw ./Engine/Programs/AutomationTool/Saved

# Enable Vulkan support for NVIDIA GPUs
USER root
RUN apt-get update && apt-get install -y --no-install-recommends libvulkan1 && \
	rm -rf /var/lib/apt/lists/* && \
	VULKAN_API_VERSION=`dpkg -s libvulkan1 | grep -oP 'Version: [0-9|\.]+' | grep -oP '[0-9|\.]+'` && \
	mkdir -p /etc/vulkan/icd.d/ && \
	echo \
	"{\
		\"file_format_version\" : \"1.0.0\",\
		\"ICD\": {\
			\"library_path\": \"libGLX_nvidia.so.0\",\
			\"api_version\" : \"${VULKAN_API_VERSION}\"\
		}\
	}" > /etc/vulkan/icd.d/nvidia_icd.json
USER ue4
