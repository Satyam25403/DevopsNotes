FROM ubuntu:20.04

# Avoid interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install openssh-server and sudo
RUN apt update && \
    apt install -y openssh-server sudo && \
    mkdir /var/run/sshd

# Create a new user
RUN useradd -m sshuser && \
    echo 'sshuser:sshpassword' | chpasswd && \
    usermod -aG sudo sshuser

# Allow root login & password auth (for simplicity, not recommended in production)
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Expose SSH port
EXPOSE 22

# Start SSH service
CMD ["/usr/sbin/sshd", "-D"]