FROM ubuntu:focal as gramine

RUN env DEBIAN_FRONTEND=noninteractive apt-get update \
    && env DEBIAN_FRONTEND=noninteractive apt-get install -y \
       autoconf \
       bison \
       build-essential \
       coreutils \
       gawk \
       git \
       libcurl4-openssl-dev \
       libprotobuf-c-dev \
       meson \
       protobuf-c-compiler \
       python3 \
       python3-pip \
       python3-protobuf \
       python3-jinja2 \
       python3-click \
       wget \
       pkg-config \
    && python3 -B -m pip install 'toml>=0.10' 'meson>=0.55'

RUN echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' \
    > /etc/apt/sources.list.d/intel-sgx.list \
    && wget https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key \
    && apt-key add intel-sgx-deb.key

RUN apt-get update \
    && apt-get install -y \
    libsgx-urts \
    libsgx-dcap-ql \
    libsgx-dcap-quote-verify \
    libsgx-dcap-default-qpl \
    libsgx-quote-ex \
    libsgx-dcap-quote-verify-dev

RUN git clone https://github.com/gramineproject/gramine.git

WORKDIR /gramine
COPY 0001-ra-tls-secret-prov-use-cert-manager-filenames.patch .
RUN GIT_COMMITTER_EMAIL=foo@example.com git am *.patch \
    && mkdir asm \
    && wget https://raw.githubusercontent.com/torvalds/linux/master/arch/x86/include/uapi/asm/sgx.h -O asm/sgx.h \
    && meson build --prefix="/gramine/meson_build_output" \
       --buildtype=release \
       -Ddirect=enabled \
       -Dsgx=enabled \
       -Ddcap=enabled \
       -Dsgx_driver_include_path=/gramine \
    && ninja -C build \
    && ninja -C build install
RUN cd CI-Examples/ra-tls-secret-prov \
    && PKG_CONFIG_PATH="/gramine/meson_build_output/lib/x86_64-linux-gnu/pkgconfig" make dcap

FROM ubuntu:focal

RUN env DEBIAN_FRONTEND=noninteractive apt-get update \
    && env DEBIAN_FRONTEND=noninteractive apt-get install -y \
        wget \
        gnupg2

RUN echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' \
    > /etc/apt/sources.list.d/intel-sgx.list \
    && wget https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key \
    && apt-key add intel-sgx-deb.key

RUN env DEBIAN_FRONTEND=noninteractive apt-get update \
    && env DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
       python3 \
       python3-pip \
       python3-protobuf \
       python3-jinja2 \
       python3-click \
       libnss-mdns \
       libnss-myhostname \
       libprotobuf-c1 \
       libsgx-urts \
       libsgx-dcap-ql \
       libsgx-quote-ex \
       libsgx-dcap-default-qpl \
       libsgx-dcap-quote-verify \
       binutils \
       git \
       make \
    && pip3 install --user torchvision pillow 'toml>=0.10'

COPY --from=gramine /gramine/meson_build_output /gramine/meson_build_output/

RUN mkdir /ra-tls-secret-prov
COPY --from=gramine /gramine/CI-Examples/ra-tls-secret-prov/secret_prov_server_dcap /ra-tls-secret-prov

RUN git clone --depth=1 https://github.com/gramineproject/examples.git
WORKDIR /examples/pytorch

ENV PATH="${PATH}:/gramine/meson_build_output/bin"
ENV LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:/gramine/meson_build_output/lib"
ENV PYTHONPATH="/gramine/meson_build_output/lib/python3.8/site-packages"

# prepare encrypted pre-trained model and input data as part of the
# image build for demo purposes. Generate files/wrap-key using gramine-sgx-pf-crypt:
# gramine-sgx-pf-crypt gen-key -w <path/wrap-key>
COPY wrap-key wrap-key
RUN python3 download-pretrained-model.py \
    && mkdir plaintext/ \
    && mv input.jpg classes.txt alexnet-pretrained.pt plaintext/ \
    && gramine-sgx-pf-crypt encrypt -w wrap-key -i plaintext/input.jpg -o input.jpg \
    && gramine-sgx-pf-crypt encrypt -w wrap-key -i plaintext/classes.txt -o classes.txt \
    && gramine-sgx-pf-crypt encrypt -w wrap-key -i plaintext/alexnet-pretrained.pt -o alexnet-pretrained.pt \
    && rm -r wrap-key plaintext/

COPY pytorch.manifest.template pytorch.manifest.template

# generate enclave signing key as part of the image build for demo purposes only.
RUN openssl genrsa -3 -out enclave-key.pem 3072 \
    && sed -i -e 's:result.txt:/mnt/pytorch/result.txt:' pytorchexample.py \
    && SGX_SIGNER_KEY=enclave-key.pem make ARCH_LIBDIR="/lib/x86_64-linux-gnu" SGX=1 \
    && rm enclave-key.pem

ENTRYPOINT ["gramine-sgx", "./pytorch", "./pytorchexample.py"]
