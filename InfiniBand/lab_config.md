# Install Software RCoE


sudo modinfo rdma_rxe
sudo modprobe rdma_rxe
sudo rdma link add rxe0 type rxe netdev lo
rdma link show
