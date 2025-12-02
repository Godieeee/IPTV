// Stremio Xtream VOD Addon (auto) - CORRIGÉ
const { addonBuilder, serveHTTP } = require('stremio-addon-sdk');
const fetch = require('node-fetch');

// ====== CONFIG (Vérifiez et remplacez les valeurs ci-dessous par les vôtres) ======
const X_HOST = 'http://ontv4tv.net:80';
const X_USERNAME = '942766655593335';
const X_PASSWORD = '1593574628';
// =================================================================================

const builder = new addonBuilder({
    id: 'org.xtream.vod.fixed',
    version: '1.0.1',
    name: 'Xtream VOD (Fixé)',
    resources: ["catalog", "meta", "stream", "manifest"],
    types: ["movie", "series"],
    catalogs: [
        { type: 'movie', id: 'xtream-movies' },
        { type: 'series', id: 'xtream-series' }
    ],
    idPrefixes: ['xtream:']
});

// Cache simple pour stocker temporairement les informations M3U et les détails des séries
const cache = {};

// Helper: Tente de récupérer et de parser une réponse JSON
async function fetchJson(url) {
    const res = await fetch(url);
    const text = await res.text();
    try { return JSON.parse(text); } catch (e) { return text; }
}

// Récupère la liste M3U complète et la met en cache (y compris les URLs et les titres)
async function fetchM3U() {
    if (cache.m3u && (Date.now() - cache.m3u.timestamp < 3600000)) { // Cache 1 heure
        return cache.m3u.items;
    }
    
    const url = `${X_HOST}/get.php?username=${X_USERNAME}&password=${X_PASSWORD}&type=m3u_plus&output=hls`;
    console.log('Fetching M3U fallback...');
    try {
        const res = await fetch(url);
        const text = await res.text();
        const lines = text.split(/\r?\n/).map(l => l.trim()).filter(Boolean);
        const items = [];
        let itemIndex = 0;
        
        for (let i = 0; i < lines.length; i++) {
            if (lines[i].startsWith('#EXTINF')) {
                const title = lines[i].split(',').slice(1).join(',').trim() || 'Unknown';
                const url = lines[i + 1] || '';
                if (url.startsWith('http')) {
                    items.push({ 
                        title: title, 
                        url: url, 
                        idx: itemIndex // Indice pour l'utiliser dans le stream handler
                    });
                    itemIndex++;
                }
            }
        }
        cache.m3u = { items, timestamp: Date.now() };
        return items;
    } catch (e) {
        console.error('M3U Fetch Error:', e);
        return [];
    }
}

// Récupère les streams VOD (films) via l'API
async function getVodStreams() {
    const url = `${X_HOST}/player_api.php?username=${X_USERNAME}&password=${X_PASSWORD}&action=get_vod_streams`;
    try {
        const data = await fetchJson(url);
        if (Array.isArray(data)) return data; 
    } catch (e) {}
    return null;
}

// Récupère la liste des séries via l'API
async function getSeriesList() {
    const url = `${X_HOST}/player_api.php?username=${X_USERNAME}&password=${X_PASSWORD}&action=get_series`;
    try {
        const data = await fetchJson(url);
        if (Array.isArray(data)) return data; 
    } catch (e) {}
    return null;
}

// ##############################################################################
// GESTIONNAIRE DE CATALOGUE (Lists des films et séries)
// ##############################################################################

builder.defineCatalogHandler(async function(args, cb) {
    try {
        let metas = [];
        
        if (args.type === 'movie') {
            const vod = await getVodStreams();
            if (vod && vod.length) {
                metas = vod.map(v => ({
                    id: `xtream:movie:${v.stream_id || v.movie_id || v.id_vod}`,
                    type: 'movie',
                    name: v.name || v.title || v.stream_name || 'Unknown',
                    poster: v.info ? v.info.movie_image : (v.stream_icon || undefined),
                    posterShape: 'poster'
                }));
            }
        } 
        
        if (args.type === 'series') {
            const series = await getSeriesList();
            if (series && series.length) {
                metas = series.map(s => ({
                    id: `xtream:series:${s.series_id || s.id || s.sid}`,
                    type: 'series',
                    name: s.name || s.series_name || 'Unknown',
                    poster: s.info ? s.info.series_image : (s.stream_icon || undefined),
                    posterShape: 'poster'
                }));
            }
        } 
        
        // Fallback M3U pour les éléments non trouvés via l'API (Ajoutés uniquement aux films pour la simplicité)
        if (metas.length === 0) {
            const m3u = await fetchM3U();
            metas = m3u.map((it, idx) => ({
                id: `xtream:m3u:${it.idx}`, // ID unique pour la lecture
                type: args.type, // Utilise le type demandé
                name: it.title,
                posterShape: 'poster'
            }));
        }

        cb(null, { metas });

    } catch (err) {
        cb(err.message);
    }
});

// ##############################################################################
// GESTIONNAIRE DE MÉTA (Détails et épisodes)
// ##############################################################################

builder.defineMetaHandler(async function(args, cb) {
    try {
        const id = args.id || '';
        const parts = id.split(':');
        const type = parts[1]; // movie, series ou m3u
        const vid = parts[2]; // ID du stream/série ou index M3U

        let meta = null;

        if (type === 'movie') {
            const url = `${X_HOST}/player_api.php?username=${X_USERNAME}&password=${X_PASSWORD}&action=get_vod_info&vod_id=${vid}`;
            const data = await fetchJson(url);
            
            if (data && data.info) {
                 meta = {
                    id,
                    type: 'movie',
                    name: data.info.name || data.info.title || `Movie ${vid}`,
                    description: data.info.plot || undefined,
                    poster: data.info.movie_image || data.info.cover || undefined,
                    background: data.info.backdrop_path ? `${X_HOST}/images/movie/${data.info.backdrop_path}` : undefined,
                    imdb_id: data.info.imdb_id || undefined,
                };
            }
        } else if (type === 'series') {
            const url = `${X_HOST}/player_api.php?username=${X_USERNAME}&password=${X_PASSWORD}&action=get_series_info&series_id=${vid}`;
            const data = await fetchJson(url);
            
            if (data && data.info) {
                const episodes = [];
                // Groupement des épisodes par saison
                if (data.episodes) {
                    for (const seasonNum in data.episodes) {
                        data.episodes[seasonNum].forEach(ep => {
                            episodes.push({
                                id: `xtream:series:${vid}:${ep.season}:${ep.episode_num || ep.episode_id}`, 
                                title: ep.title || `Episode ${ep.episode_num}`,
                                season: ep.season,
                                episode: ep.episode_num,
                                overview: ep.info ? ep.info.plot : undefined,
                                thumbnail: ep.info ? ep.info.screenshot : undefined,
                                released: new Date(ep.air_date).toISOString(),
                                stream_id: ep.id // Ajout pour le stream handler
                            });
                        });
                    }
                }

                meta = {
                    id,
                    type: 'series',
                    name: data.info.name || `Series ${vid}`,
                    description: data.info.plot || undefined,
                    poster: data.info.cover || undefined,
                    background: data.info.backdrop_path ? `${X_HOST}/images/series/${data.info.backdrop_path}` : undefined,
                    videos: episodes, // Les épisodes sont stockés dans la propriété 'videos' pour Stremio
                };
            }
        } else if (type === 'm3u') {
            const m3uItems = await fetchM3U();
            const item = m3uItems[parseInt(vid)];
            if (item) {
                 meta = {
                    id,
                    type: 'movie', 
                    name: item.title,
                    description: 'Stream via M3U Fallback. Le titre peut être moins précis.',
                    posterShape: 'poster'
                };
            }
        }

        cb(null, { meta });

    } catch (err) {
        cb(err.message);
    }
});


// ##############################################################################
// GESTIONNAIRE DE FLUX (Stream)
// ##############################################################################

builder.defineStreamHandler(async function(args, cb) {
    try {
        const id = args.id || '';
        const parts = id.split(':');
        const type = parts[1];
        const streams = [];

        if (type === 'movie') {
            const vid = parts[2];
            // Format standard Xtream pour les films
            const streamUrl = `${X_HOST}/movie/${X_USERNAME}/${X_PASSWORD}/${vid}.mp4`;
            streams.push({ 
                title: 'Xtream VOD', 
                url: streamUrl,
                name: 'Direct MP4/HLS'
            });
            
        } else if (type === 'series' && parts.length >= 4) {
            // ID de série: xtream:series:SERIES_ID:SEASON_NUM:EPISODE_NUM
            const seriesId = parts[2];
            const seasonNum = parts[3];
            const episodeNum = parts[4];
            
            // On doit refetch l'info de la série pour trouver l'ID du stream de l'épisode
            const seriesUrl = `${X_HOST}/player_api.php?username=${X_USERNAME}&password=${X_PASSWORD}&action=get_series_info&series_id=${seriesId}`;
            const data = await fetchJson(seriesUrl);

            let episodeStreamId = null;

            if (data && data.episodes && data.episodes[seasonNum]) {
                const episode = data.episodes[seasonNum].find(ep => 
                    String(ep.episode_num) === String(episodeNum) || 
                    String(ep.id) === String(episodeNum) // Certaines APIs utilisent l'ID stream à la place du numéro
                );

                if (episode) {
                    episodeStreamId = episode.id || episode.stream_id;
                }
            }

            if (episodeStreamId) {
                // Format standard Xtream pour les séries/épisodes
                const streamUrl = `${X_HOST}/series/${X_USERNAME}/${X_PASSWORD}/${episodeStreamId}.mp4`;
                 streams.push({ 
                    title: `S${seasonNum} E${episodeNum}`, 
                    url: streamUrl,
                    name: 'Direct MP4/HLS'
                });
            }

        } else if (type === 'm3u') {
            const m3uIndex = parseInt(parts[2]);
            const m3uItems = await fetchM3U();
            const item = m3uItems.find(i => i.idx === m3uIndex);
            
            if (item) {
                 streams.push({ 
                    title: item.title, 
                    url: item.url,
                    name: 'M3U Stream'
                });
            }
        }

        cb(null, { streams });

    } catch (err) {
        cb(err.message);
    }
});


// ##############################################################################
// DÉMARRAGE DU SERVEUR
// ##############################################################################

// Le SDK gère le manifeste et le routing automatiquement avec la fonction 'serveHTTP'
// Plus besoin de définir un router Express manuel pour le manifeste.

serveHTTP(builder.get "}
