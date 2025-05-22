public class GeocodeResult {
    private String networkId;
    private String address;
    private String city;
    private Double lat;
    private Double lng;
    private String result; // SUCCESS / QPS_LIMIT_EXCEEDED / NO_RESULT / ...

    public static GeocodeResult success(String address, String city, Double lat, Double lng) {
        GeocodeResult r = new GeocodeResult();
        r.address = address;
        r.city = city;
        r.lat = lat;
        r.lng = lng;
        r.result = "SUCCESS";
        return r;
    }
    
    public static GeocodeResult failure(String address, String reason) {
        GeocodeResult r = new GeocodeResult();
        r.address = address;
        r.result = reason;
        return r;
    }
    
    public void setNetworkId(String networkId) {
        this.networkId = networkId;
    }
    
    @Override
    public String toString() {
        return "[networkId=" + networkId + ", address=" + address + ", result=" + result + "]";
    }
}