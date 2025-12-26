FROM grafana/grafana:8.1.6
#12.4.0-20012734117

#RUN grafana-cli plugins install grafana-piechart-panel

ADD ./config.ini /etc/grafana/config.ini
ADD ./provisioning /etc/grafana/provisioning
ADD ./grafana-piechart-panel /etc/grafana/grafana-piechart-panel
ADD ./dashboards /var/lib/grafana/dashboards
